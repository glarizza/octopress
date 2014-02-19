---
layout: post
title: "Building a Functional Puppet Workflow Part 3: Dynamic Environments with R10k"
date: 2014-02-18 09:48:18 -0800
comments: true
categories: ['r10k', 'puppet', 'workflow', 'environments']
---

Workflows are like kickball games: everyone knows the general idea of what's
going on, there's an orderly progression towards an end-goal, nobody wants to
be excluded, and people lose their shit when they get hit in the face by a big
rubber ball.  Okay, so maybe it's not a perfect mapping but you get the idea.

The previous two posts ([one][firstpost] and [two][secondpost]) focused on
writing modules, wrapping modules, and classification. While BOTH of these
things are very important in the grand scheme of things, one of the biggest
problems people get hung-up on is how do you iterate upon your modules, and,
more importantly, how do you eventually get these changes pushed into production
in a reasonably orderly fashion?

This post is going to be all over the place. We're gonna cover the idea of
separate environments in Puppet, touch on dynamic environments, and round it
out with that mother-of-a-shell-script-turned-personal-savior, R10k. Hold on
to your shit.

## Puppet Environments

[Puppet has the concept of 'environments'][puppetenv] where you can logically
separate your modules and manifest (read: `site.pp`) into separate folders
to allow for nodes to get entirely separate bits of code based on which
'environment' the node belongs to.

Puppet environments are statically set in `puppet.conf`, but,
[as other blog posts have noted][puppetgit], you can do some crafty things in
`puppet.conf` to give you the solution of having 'dynamic environments'.

**NOTE**: The solutions in this post are going to rely on Puppet environments,
however environments aren't without their own shortcomings
[namely, this bug on Ruby plugins in Puppet][envbug]). For testing and
promoting Puppet classes written in the DSL, environments will help you out
greatly. For complete separation of Ruby instances and any plugins to Puppet
written in Ruby, however, you'll need separate masters (which is something that
I won't be covering in this article).

## One step further - 'dynamic' environments

Adrien Thebo, hitherto known as 'Finch', - who is known for building awesome
things and talking like he's fresh from a Redbull binge - created the
[now-famous blog post on creating dynamic environments in Puppet with git.][puppetgit]
That post relied upon a post-commit hook to do all the jiggery-pokery necessary
to checkout the correct branches in the correct places, and thus it had a heavy
reliance upon git.

Truly, the only magic in `puppet.conf` was the inclusion of '$environment' in
the `modulepath` configuration entry on the Puppet master (literally that
string and not the evaluated form of your environment). By doing that, the
Puppet master would replace the string '$environment' with the environment
of the node checking in and would look to that path for Puppet manifests
and modules. If you use something OTHER than git, it would be up to you to
create a post-receive hook that populated those paths, but you could still
replicate the results (albiet with a little work on your part).

People used this pattern and it worked fairly well. Hell, it STILL works
fairly well, nothing has changed to STOP you from using it. What changed,
however, was the ecosystem around modules, the need for individual module
testing, and the further need to automate this whole goddamn process.

Before we deliver the 'NEW SOLUTION', let's provide a bit of history and
context.

## Module repositories: the one-to-many problem

I [touched on this topic in the first post][firstpost], but one of the first
problems you encounter when putting your modules in version control is whether
or not to have ONE GIANT REPO with all of your modules, or a repository for
every module you create. In the past we recommended putting every module in
one repository (namely because it was easier, the module sharing landscape
was pretty barren, and teams were smaller). Now, we recommend the opposite
for the following reasons:

* Individual repos mean individual module development histories
* Most VCS solutions don't have per-folder ACLs for a single repositories;
having multiple repos allows per-module security settings.
* With the one-repository-per-module solution, modules you pull down from the
Forge (or Github) must be committed to your repo. Having multiple
repositories for each module allow you to keep everything separate
* Publishing this module to the Forge (or Github/Stash/whatever) is easier
with separate repos (rather than having to split-out the module later).

The problem with having a repository for every Puppet module you create is that
you need a way to map every module with every Puppet master (and, also which
version of every module should be installed in which Puppet environment).

[A project called librarian-puppet][librarian] sprang up that created the
'`Puppetfile`', a file that would map modules and their versions to a
specific directory. Librarian was awesome,
[but, as Finch noted in his post][finchlibrarian], it had some shortcomings
when used in an environment with many and fast-changing modules.
[His solution, that he documented here,][finchr10k], was the tool we now come
to know as R10k.

## Enter R10k

R10k is essentially a Ruby project that wraps a bunch of shell commands you
would NORMALLY use to maintain an environment of ever-changing Puppet modules.
Its power is in its ability to use Git branches combined with a `Puppetfile`
to keep your Puppet environments in-sync. Because of this, R10k is CURRENTLY
restricted to git. There have been rumblings of porting it to Hg or svn, but
I know of no serious attempts at doing this (and if you ARE doing this, may
god have mercy on your soul). Great, so how does it work?

Well, you'll need one main repository SIMPLY for tracking the `Puppetfile`.
[I've got one right here,][mainrepo] and it only has my `Puppetfile` and a
`site.pp` file for classification (should you use it).

**NOTE**: The Puppetfile and librarian-puppet-like capabilities under the hood
are going to be doing most of the work here - this repository is solely so you
can create topic branches with changes to your `Puppetfile` that will
eventually become dynamically-created Puppet environments.

Let's take a look at the `Puppetfile` and see what's going on:

{% codeblock lang:ruby Puppetfile %}
forge "http://forge.puppetlabs.com"

# Modules from the Puppet Forge
mod "puppetlabs/stdlib"
mod "puppetlabs/apache", "0.11.0"
mod "puppetlabs/pe_gem"
mod "puppetlabs/mysql"
mod "puppetlabs/firewall"
mod "puppetlabs/vcsrepo"
mod "puppetlabs/git"
mod "puppetlabs/inifile"
mod "zack/r10k"
mod "gentoo/portage"
mod "thias/vsftpd"


# Modules from Github using various references
mod "wordpress",
  :git => "git://github.com/hunner/puppet-wordpress.git",
  :ref => '0.4.0'

mod "property_list_key",
  :git => "git://github.com/glarizza/puppet-property_list_key.git",
  :ref => '952a65d9ea2c5809f4e18f30537925ee45548abc'

mod 'redis',
  :git => 'git://github.com/glarizza/puppet-redis',
  :ref => 'feature/debian_support'
{% endcodeblock %}

This example lists the syntax for dealing with modules from both the
Forge and Github, as well as pulling specific versions of modules (whether
versions in the case of the Forge, or Github references as tags, branches,
or even specific commits). The syntax is not hard to follow - just remember
that we're mapping modules and their versions to a set/known environment.

For every topic branch on this repository (containing the `Puppetfile`), R10k
will in turn create a Puppet environment with the same name. For this reason,
it's convention to rename the 'master' branch to 'production' since that's the
default environment in Puppet (note that renaming branches locally is easy -
renaming the branch on Github can sometimes be a pain in the ass). You will
also note why it's going to be somewhat hard to map R10k to subversion, for
example, due to the lack of lightweight branching schemes.

To explain any more of R10k reads just as if I were describing its installation,
so let's quit screwing around and actually INSTALL/SETUP the damn thing.

## Setting up R10k

As I mentioned before, we [have the main repository][mainrepo] that will be
used to track the `Puppetfile`, which in turn will track the modules to
be installed (whether from The Forge, Github, or some internal git repo). Like
any good Puppet component, R10k itself can be setup with a Puppet module.
[The module I'll be using][forgemod] was developed by [Zack Smith][zack],
and is pretty simple to get started. Let's download it from the forge first:

```
[root@master1 vagrant]# puppet module install zack/r10k
Notice: Preparing to install into /etc/puppetlabs/puppet/modules ...
Notice: Downloading from https://forge.puppetlabs.com ...
Notice: Installing -- do not interrupt ...
/etc/puppetlabs/puppet/modules
└─┬ zack-r10k (v1.0.2)
  ├─┬ gentoo-portage (v2.1.0)
  │ └── puppetlabs-concat (v1.0.1)
  ├── mhuffnagle-make (v0.0.2)
  ├── puppetlabs-gcc (v0.1.0)
  ├── puppetlabs-git (v0.0.3)
  ├── puppetlabs-inifile (v1.0.1)
  ├── puppetlabs-pe_gem (v0.0.1)
  ├── puppetlabs-ruby (v0.1.0)
  └── puppetlabs-vcsrepo (v0.2.0)
```

The module will be installed into the first path in your modulepath, which in
the case above is `/etc/puppetlabs/puppet/modules`. This modulepath will change
due to the way we're going to setup our dynamic Puppet environments. For this
example, I'm going to have environments dynamically generated at
`/etc/puppetlabs/puppet/environments`, so let's create that directory first:

```
[root@master1 vagrant]# mkdir -p /etc/puppetlabs/puppet/environments
```

Now, we need to setup R10k on this machine. The module we downloaded will
allow us to do that, but we'll need to create a small Puppet manifest that
will allow us to setup R10k out-of-band from a regular Puppet run (you CAN
continuously-enforce R10k configuration in-band with your regular Puppet
run, but if we're setting up a Puppet master to use R10k to serve out dynamic
environments it's possible to create a chicken-and-egg situation.). Let's
generate a file called `r10k_installation.pp` in `/var/tmp` and have it look
like the following:

{% codeblock lang:puppet /var/tmp/r10k_installation.pp %}
class { 'r10k':
  version           => '1.1.3',
  sources           => {
    'puppet' => {
      'remote'  => 'https://github.com/glarizza/puppet_repository.git',
      'basedir' => "${::settings::confdir}/environments",
      'prefix'  => false,
    }
  },
  purgedirs         => ["${::settings::confdir}/environments"],
  manage_modulepath => true,
  modulepath        => "${::settings::confdir}/environments/\$environment/modules:/opt/puppet/share/puppet/modules",
}
{% endcodeblock %}

So what is every section of that declaration doing?

* **`version => '1.1.3'`** sets the version of the R10k gem to install
* **`sources => {...}`** is a hash of sources that R10k is going to track. For
now it's only our [main Puppet repo][mainrepo], but you can also track a Hiera
installation too. This hash accepts key/value pairs for configuration settings
that are going to be written to `/etc/r10k.yaml`, which is R10k's main
configuration file. The keys in-use are `remote`, which is the path to the
repository to-be-checked-out by R10k, `basedir`, which is the path on-disk
to where dynamic environments are to be created (we're using the
`$::settings::confdir` variable which maps to the Puppet master's configuration
directory, or `/etc/puppetlabs/puppet`), and `prefix` which is a boolean
to determine whether to use R10k's source-prefixing feature. **NOTE: the `false`
value is a BOOLEAN value, and thus SHOULD NOT BE QUOTED. Quoting it turns it
into a string, which matches as a boolean TRUE value. Don't quote `false` -
that's bad, mmkay.**
* **`purgedirs=> ["${::settings::confdir}/environments"]`** is configuring R10k
to implement purging on the environments directory (so any folders that R10k
doesn't create it will delete). This configuration MAY be moot with newer
versions of R10k as I believe it implements this behavior by default.
* **`manage_modulepath => true`** will ensure that this module sets the `modulepath`
configuration item in `/etc/puppetlabs/puppet/puppet.conf`
* **`modulepath => ...`** sets the `modulepath` value to be dropped into
`/etc/puppetlabs/puppet/puppet.conf`. Note that we are interpolating variables
(`$::settings::confdir` again), AND inserting the LITERAL string of `$environment`
into the `modulepath` - this is because Puppet will replace `$environment` with
the value of the agent's environment at catalog compilation.

**JUST IN CASE YOU MISSED IT: Don't quote the `false` value for the prefix setting
in the `sources` block. That is all.**

Okay, we have our one-time Puppet manifest, and now the only thing left to do
is to run it:

```
[root@master1 tmp]# puppet apply /var/tmp/r10k_installation.pp
Notice: Compiled catalog for master1 in environment production in 2.05 seconds
Notice: /Stage[main]/R10k::Config/File[r10k.yaml]/ensure: defined content as '{md5}0b619d5148ea493e2d6a5bb205727f0c'
Notice: /Stage[main]/R10k::Config/Ini_setting[R10k Modulepath]/value: value changed '/etc/puppetlabs/puppet/modules:/opt/puppet/share/puppet/modules' to '/etc/puppetlabs/puppet/environments/$environment/modules:/opt/puppet/share/puppet/modules'
Notice: /Package[r10k]/ensure: created
Notice: /Stage[main]/R10k::Install::Pe_gem/File[/usr/bin/r10k]/ensure: created
Notice: Finished catalog run in 10.55 seconds
```

At this point, it goes without saying that `git` needs to be installed, but
if you're firing up a new VM that DOESN'T have `git`, then R10k is going to
spit out an awesome error - so ensure that `git` is installed.  After that,
let's synchronize R10k with the `r10k deploy environment -pv` command (`-p`
for `Puppetfile` synchronization and `-v` for verbose mode):

```
[root@master1 puppet]# r10k deploy environment -pv
[R10K::Task::Deployment::DeployEnvironments - INFO] Loading environments from all sources
[R10K::Task::Environment::Deploy - NOTICE] Deploying environment production
[R10K::Task::Puppetfile::Sync - INFO] Loading modules from Puppetfile into queue
[R10K::Task::Module::Sync - INFO] Deploying redis into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying property_list_key into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying wordpress into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying r10k into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying make into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying concat into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying vsftpd into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying portage into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying r10k into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying inifile into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying git into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying vcsrepo into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying firewall into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying ruby into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying mysql into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying pe_gem into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying apache into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying stdlib into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying redis into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying property_list_key into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying wordpress into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying r10k into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying make into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying concat into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying vsftpd into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying portage into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying r10k into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying inifile into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying git into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying vcsrepo into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying firewall into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying ruby into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying mysql into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying pe_gem into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying apache into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying stdlib into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Environment::Deploy - NOTICE] Deploying environment master
[R10K::Task::Puppetfile::Sync - INFO] Loading modules from Puppetfile into queue
[R10K::Task::Module::Sync - INFO] Deploying redis into /etc/puppetlabs/puppet/environments/master/modules
[R10K::Task::Module::Sync - INFO] Deploying property_list_key into /etc/puppetlabs/puppet/environments/master/modules
[R10K::Task::Module::Sync - INFO] Deploying wordpress into /etc/puppetlabs/puppet/environments/master/modules
[R10K::Task::Module::Sync - INFO] Deploying vsftpd into /etc/puppetlabs/puppet/environments/master/modules
[R10K::Task::Module::Sync - INFO] Deploying portage into /etc/puppetlabs/puppet/environments/master/modules
[R10K::Task::Module::Sync - INFO] Deploying r10k into /etc/puppetlabs/puppet/environments/master/modules
[R10K::Task::Module::Sync - INFO] Deploying inifile into /etc/puppetlabs/puppet/environments/master/modules
[R10K::Task::Module::Sync - INFO] Deploying git into /etc/puppetlabs/puppet/environments/master/modules
[R10K::Task::Module::Sync - INFO] Deploying vcsrepo into /etc/puppetlabs/puppet/environments/master/modules
[R10K::Task::Module::Sync - INFO] Deploying firewall into /etc/puppetlabs/puppet/environments/master/modules
[R10K::Task::Module::Sync - INFO] Deploying mysql into /etc/puppetlabs/puppet/environments/master/modules
[R10K::Task::Module::Sync - INFO] Deploying pe_gem into /etc/puppetlabs/puppet/environments/master/modules
[R10K::Task::Module::Sync - INFO] Deploying apache into /etc/puppetlabs/puppet/environments/master/modules
[R10K::Task::Module::Sync - INFO] Deploying stdlib into /etc/puppetlabs/puppet/environments/master/modules
[R10K::Task::Module::Sync - INFO] Deploying redis into /etc/puppetlabs/puppet/environments/master/modules
[R10K::Task::Module::Sync - INFO] Deploying property_list_key into /etc/puppetlabs/puppet/environments/master/modules
[R10K::Task::Module::Sync - INFO] Deploying wordpress into /etc/puppetlabs/puppet/environments/master/modules
[R10K::Task::Module::Sync - INFO] Deploying vsftpd into /etc/puppetlabs/puppet/environments/master/modules
[R10K::Task::Module::Sync - INFO] Deploying portage into /etc/puppetlabs/puppet/environments/master/modules
[R10K::Task::Module::Sync - INFO] Deploying r10k into /etc/puppetlabs/puppet/environments/master/modules
[R10K::Task::Module::Sync - INFO] Deploying inifile into /etc/puppetlabs/puppet/environments/master/modules
[R10K::Task::Module::Sync - INFO] Deploying git into /etc/puppetlabs/puppet/environments/master/modules
[R10K::Task::Module::Sync - INFO] Deploying vcsrepo into /etc/puppetlabs/puppet/environments/master/modules
[R10K::Task::Module::Sync - INFO] Deploying firewall into /etc/puppetlabs/puppet/environments/master/modules
[R10K::Task::Module::Sync - INFO] Deploying mysql into /etc/puppetlabs/puppet/environments/master/modules
[R10K::Task::Module::Sync - INFO] Deploying pe_gem into /etc/puppetlabs/puppet/environments/master/modules
[R10K::Task::Module::Sync - INFO] Deploying apache into /etc/puppetlabs/puppet/environments/master/modules
[R10K::Task::Module::Sync - INFO] Deploying stdlib into /etc/puppetlabs/puppet/environments/master/modules
[R10K::Task::Environment::Deploy - NOTICE] Deploying environment development
[R10K::Task::Puppetfile::Sync - INFO] Loading modules from Puppetfile into queue
[R10K::Task::Module::Sync - INFO] Deploying r10k into /etc/puppetlabs/puppet/environments/development/modules
[R10K::Task::Module::Sync - INFO] Deploying property_list_key into /etc/puppetlabs/puppet/environments/development/modules
[R10K::Task::Module::Sync - INFO] Deploying wordpress into /etc/puppetlabs/puppet/environments/development/modules
[R10K::Task::Module::Sync - INFO] Deploying inifile into /etc/puppetlabs/puppet/environments/development/modules
[R10K::Task::Module::Sync - INFO] Deploying vsftpd into /etc/puppetlabs/puppet/environments/development/modules
[R10K::Task::Module::Sync - INFO] Deploying firewall into /etc/puppetlabs/puppet/environments/development/modules
[R10K::Task::Module::Sync - INFO] Deploying mysql into /etc/puppetlabs/puppet/environments/development/modules
[R10K::Task::Module::Sync - INFO] Deploying pe_gem into /etc/puppetlabs/puppet/environments/development/modules
[R10K::Task::Module::Sync - INFO] Deploying apache into /etc/puppetlabs/puppet/environments/development/modules
[R10K::Task::Module::Sync - INFO] Deploying stdlib into /etc/puppetlabs/puppet/environments/development/modules
[R10K::Task::Module::Sync - INFO] Deploying r10k into /etc/puppetlabs/puppet/environments/development/modules
[R10K::Task::Module::Sync - INFO] Deploying property_list_key into /etc/puppetlabs/puppet/environments/development/modules
[R10K::Task::Module::Sync - INFO] Deploying wordpress into /etc/puppetlabs/puppet/environments/development/modules
[R10K::Task::Module::Sync - INFO] Deploying inifile into /etc/puppetlabs/puppet/environments/development/modules
[R10K::Task::Module::Sync - INFO] Deploying vsftpd into /etc/puppetlabs/puppet/environments/development/modules
[R10K::Task::Module::Sync - INFO] Deploying firewall into /etc/puppetlabs/puppet/environments/development/modules
[R10K::Task::Module::Sync - INFO] Deploying mysql into /etc/puppetlabs/puppet/environments/development/modules
[R10K::Task::Module::Sync - INFO] Deploying pe_gem into /etc/puppetlabs/puppet/environments/development/modules
[R10K::Task::Module::Sync - INFO] Deploying apache into /etc/puppetlabs/puppet/environments/development/modules
[R10K::Task::Module::Sync - INFO] Deploying stdlib into /etc/puppetlabs/puppet/environments/development/modules
[R10K::Task::Deployment::PurgeEnvironments - INFO] Purging stale environments from /etc/puppetlabs/puppet/environments
```

I ran this first synchronization with verbose mode so you can see exactly what's
getting copied where. Futher synchronizations don't have to be in verbose mode,
but it's good for debugging. After all of that, we have an
`/etc/puppetlabs/puppet/environments` folder containing our dynamic Puppet
environments based off of the branches of the [main Puppet repo][mainrepo]:

```
[root@master1 puppet]# ls -lah /etc/puppetlabs/puppet/environments/
total 20K
drwxr-xr-x 5 root root 4.0K Feb 19 11:44 .
drwxr-xr-x 7 root root 4.0K Feb 19 11:25 ..
drwxr-xr-x 4 root root 4.0K Feb 19 11:44 development
drwxr-xr-x 5 root root 4.0K Feb 19 11:43 master
drwxr-xr-x 5 root root 4.0K Feb 19 11:42 production

[root@master1 puppet]# cd /etc/puppetlabs/puppet/environments/production/
[root@master1 production]# git branch -a
  master
* production
  remotes/origin/HEAD -> origin/master
  remotes/origin/development
  remotes/origin/master
  remotes/origin/production
```

As you can see (at the time of this writing), my [main Puppet repo][mainrepo]
has three main branches: `development`, `master`, and `production`, and so
R10k created three Puppet environments matching those names. It's somewhat
of a convention to rename the master branch to `production`, but in this
case I left it alone to demonstrate how this works.

**ONE OTHER BIG GOTCHA**: R10k does NOT resolve dependencies, and
so it is UP TO YOU to track them in your Puppetfile.  Check this out:

```
[root@master1 production]# puppet module list
Warning: Module 'puppetlabs-firewall' (v1.0.0) fails to meet some dependencies:
  'puppetlabs-puppet_enterprise' (v3.1.0) requires 'puppetlabs-firewall' (v0.3.x)
Warning: Module 'puppetlabs-stdlib' (v4.1.0) fails to meet some dependencies:
  'puppetlabs-pe_accounts' (v2.0.1) requires 'puppetlabs-stdlib' (v3.2.x)
  'puppetlabs-pe_mcollective' (v0.1.14) requires 'puppetlabs-stdlib' (v3.2.x)
  'puppetlabs-puppet_enterprise' (v3.1.0) requires 'puppetlabs-stdlib' (v3.2.x)
  'puppetlabs-request_manager' (v0.0.10) requires 'puppetlabs-stdlib' (v3.2.x)
Warning: Missing dependency 'cprice404-inifile':
  'puppetlabs-pe_puppetdb' (v0.0.11) requires 'cprice404-inifile' (>=0.9.0)
  'puppetlabs-puppet_enterprise' (v3.1.0) requires 'cprice404-inifile' (v0.10.x)
  'puppetlabs-puppetdb' (v1.5.1) requires 'cprice404-inifile' (>= 0.10.3)
Warning: Missing dependency 'puppetlabs-concat':
  'puppetlabs-apache' (v0.11.0) requires 'puppetlabs-concat' (>= 1.0.0)
  'gentoo-portage' (v2.1.0) requires 'puppetlabs-concat' (v1.0.x)
Warning: Missing dependency 'puppetlabs-gcc':
  'zack-r10k' (v1.0.2) requires 'puppetlabs-gcc' (>= 0.0.3)
/etc/puppetlabs/puppet/environments/production/modules
├── gentoo-portage (v2.1.0)
├── mhuffnagle-make (v0.0.2)
├── property_list_key (???)
├── puppetlabs-apache (v0.11.0)
├── puppetlabs-firewall (v1.0.0)  invalid
├── puppetlabs-git (v0.0.3)
├── puppetlabs-inifile (v1.0.1)
├── puppetlabs-mysql (v2.2.1)
├── puppetlabs-pe_gem (v0.0.1)
├── puppetlabs-ruby (v0.1.0)
├── puppetlabs-stdlib (v4.1.0)  invalid
├── puppetlabs-vcsrepo (v0.2.0)
├── redis (???)
├── ripienaar-concat (v0.2.0)
├── thias-vsftpd (v0.2.0)
├── wordpress (???)
└── zack-r10k (v1.0.2)
/opt/puppet/share/puppet/modules
├── cprice404-inifile (v0.10.3)
├── puppetlabs-apt (v1.1.0)
├── puppetlabs-auth_conf (v0.1.7)
├── puppetlabs-firewall (v0.3.0)  invalid
├── puppetlabs-java_ks (v1.1.0)
├── puppetlabs-pe_accounts (v2.0.1)
├── puppetlabs-pe_common (v0.1.0)
├── puppetlabs-pe_mcollective (v0.1.14)
├── puppetlabs-pe_postgresql (v0.0.5)
├── puppetlabs-pe_puppetdb (v0.0.11)
├── puppetlabs-postgresql (v2.5.0)
├── puppetlabs-puppet_enterprise (v3.1.0)
├── puppetlabs-puppetdb (v1.5.1)
├── puppetlabs-reboot (v0.1.2)
├── puppetlabs-request_manager (v0.0.10)
├── puppetlabs-stdlib (v3.2.0)  invalid
└── ripienaar-concat (v0.2.0)
```

I've installed Puppet Enterprise 3.1.0, and so `/opt/puppet/share/puppet/modules`
reflects the state of the Puppet Enterprise (also known as 'PE') modules at that
time. You can see that there are some conflicts because certain modules require
certain versions of other modules. This is currently the nature of the beast
with regard to Puppet modules. Some of these errors are loud and incidental
(i.e. someone set a dependency on a version and forgot to update it), some
are due to namespace changes (i.e. `cfprice-inifile` being ported over to
`puppetlabs-inifile`), and so on. Basically, ensure that you handle the
dependencies you care about inside the `Puppetfile` as R10k won't do it for you.  

There - we've done it!  We've configured R10k!  Now how the hell do you use it?

## R10k demonstration - from module iteration to environment iteration

Let's take the environment we've setup in the previous steps and walk you
through adding a new module to your production environment, iterating upon
that module, pushing the changes to that module, pushing the changes to a
Puppet environment, and then promoting those changes to production.

**NOTES ON THE SETUP OF THIS DEMO:**

* In this demonstration, classification method is going to be left to the user
(i.e. it's not a part of the magic).  So, when I tell you to classify your
node with a specific class, I don't care if you use the Puppet Enterprise
Console, `site.pp`, or any other manner.
* I'm using Github for my repositories so that you folk watching and playing
along at home can have something to follow. Feel free to substitute Github
for something like Atlassian Stash/Bitbucket, internal repos, or whatever.

### Add the module to an environment

The module we'll be working with, [a simple module called 'notifyme'][notifyme],
will notify a message that will help us track the module's process through all
phases of iteration.

The first thing we need to do is to add the module to an environment, so let's
dynamically create a NEW environment by creating a new topic branch and pushing
it up to [the main puppet repo][mainrepo]. I will perform this step on my laptop
and outside of the VM I'm using to test R10k:

```
└(~/src/puppet_repository)▷ git branch
  master
* production

└(~/src/puppet_repository)▷ git checkout -b notifyme
Switched to a new branch 'notifyme'

└(~/src/puppet_repository)▷ vim Puppetfile

# Perform the changes to Puppetfile here

└(~/src/puppet_repository)▷ git add Puppetfile
└(~/src/puppet_repository)▷ git commit
[notifyme 5239538] Add the 'notifyme' module
 1 file changed, 3 insertions(+)

└(~/src/puppet_repository)▷ git push origin notifyme:notifyme
Counting objects: 5, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 348 bytes, done.
Total 3 (delta 1), reused 0 (delta 0)
To https://github.com/glarizza/puppet_repository.git
 * [new branch]      notifyme -> notifyme
```

The contents I added to my `Puppetfile` look like this:

{% codeblock lang:ruby Puppetfile %}
mod "notifyme",
  :git => "git://github.com/glarizza/puppet-notifyme.git"
{% endcodeblock %}

### Perform an R10k synchronization

To pull the new dynamic environment down to the Puppet master, do another
R10k synchronization with `r10k deploy environment -pv`:

```
[root@master1 production]# r10k deploy environment -pv
[R10K::Task::Deployment::DeployEnvironments - INFO] Loading environments from all sources
[R10K::Task::Environment::Deploy - NOTICE] Deploying environment production
<snip for brevity>
[R10K::Task::Environment::Deploy - NOTICE] Deploying environment notifyme
[R10K::Task::Puppetfile::Sync - INFO] Loading modules from Puppetfile into queue
[R10K::Task::Module::Sync - INFO] Deploying notifyme into /etc/puppetlabs/puppet/environments/notifyme/modules
<more snipping>
```

I only included the relevant messages, but you can see that it pulled in a new
environment called 'notifyme' that ALSO pulled in a module called 'notifyme'

### Rename the branch to avoid confusion

Suddenly I realize that this may get confusing having both an environment called
'notifyme' with a module/class called 'notifyme'. No worries, how about we rename
that branch?

```
└(~/src/puppet_repository)▷ git branch -m notifyme garysawesomeenvironment

└(~/src/puppet_repository)▷ git push origin :notifyme
To https://github.com/glarizza/puppet_repository.git
 - [deleted]         notifyme

└(~/src/puppet_repository)▷ git push origin garysawesomeenvironment:garysawesomeenvironment
Counting objects: 5, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 348 bytes, done.
Total 3 (delta 1), reused 0 (delta 0)
To https://github.com/glarizza/puppet_repository.git
 * [new branch]      garysawesomeenvironment -> garysawesomeenvironment
```

That bit of `git` renamed the 'notifyme' branch to 'garysawesomeenvironment'.
The next git command is a bit tricky - when you `git push` to a remote, it's
supposed to be:

**`git push name_of_origin local_branch_name:remote_branch_name`**

In our case, the name of our origin is LITERALLY 'origin', but we actually want
to DELETE a remote branch. The way to delete a local branch is with
`git branch -d branch_name`, but the way to delete a REMOTE branch is to push
**NOTHING** to it.  So consider the following command:

**`git push origin :notifyme`**

We're pushing to the origin named 'origin', but providing NO local branch name
and pushing that bit of nothing to the remote branch of 'notifyme'. This
kills (deletes) the remote branch.

Finally, we push to our origin named 'origin' again and push the contents of
the local branch 'garysawesomeenvironment' to the remote branch of
'garysawesomeenvironment' which in turn CREATES that branch if it doesn't exist.
Whew.  Let's run another damn synchronization:

```
[root@master1 production]# `r10k deploy environment -pv`
[R10K::Task::Deployment::DeployEnvironments - INFO] Loading environments from all sources
<more snippage>
[R10K::Task::Environment::Deploy - NOTICE] Deploying environment garysawesomeenvironment
[R10K::Task::Puppetfile::Sync - INFO] Loading modules from Puppetfile into queue
[R10K::Task::Module::Sync - INFO] Deploying notifyme into /etc/puppetlabs/puppet/environments/garysawesomeenvironment/modules
<more of that snipping shit>
R10K::Task::Deployment::PurgeEnvironments - INFO] Purging stale environments from /etc/puppetlabs/puppet/environments
```

Cool, let's check out our `environments` folder on our VM:

```
[root@master1 production]# ls -lah /etc/puppetlabs/puppet/environments/
total 24K
drwxr-xr-x 6 root root 4.0K Feb 19 13:34 .
drwxr-xr-x 7 root root 4.0K Feb 19 12:09 ..
drwxr-xr-x 4 root root 4.0K Feb 19 11:44 development
drwxr-xr-x 5 root root 4.0K Feb 19 13:33 garysawesomeenvironment
drwxr-xr-x 5 root root 4.0K Feb 19 11:43 master
drwxr-xr-x 5 root root 4.0K Feb 19 11:42 production

[root@master1 production]# cd /etc/puppetlabs/puppet/environments/garysawesomeenvironment/

[root@master1 garysawesomeenvironment]# git branch
* garysawesomeenvironment
  master
```


### Run Puppet to test the new environment

Perfect!  Now classify your node to include the 'notifyme' class, and let's run
Puppet to see what we get when we try to join the environment called
'garysawesomeenvronment':

```
[root@master1 garysawesomeenvironment]# puppet agent -t --environment garysawesomeenvironment
Info: Retrieving plugin
<snipping facts loading for brevity>
Info: Caching catalog for master1
Info: Applying configuration version '1392845863'
Notice: This is the notifyme module and its master branch
Notice: /Stage[main]/Notifyme/Notify[This is the notifyme module and its master branch]/message: defined 'message' as 'This is the notifyme module and its master branch'
Notice: Finished catalog run in 11.10 seconds
```

Cool!  Now let's try to run Puppet with another environment, say 'production':

```
[root@master1 garysawesomeenvironment]# puppet agent -t --environment production
Info: Retrieving plugin
<snipping facts loading for brevity>
Error: Could not retrieve catalog from remote server: Error 400 on SERVER: Could not find class notifyme for master1 on node master1
Warning: Not using cache on failed catalog
Error: Could not retrieve catalog; skipping run
```

We get an error because that module hasn't been loaded by R10k for that
environment.

### Tie a module version to an environment

Okay, so we added a module to a new environment, but what if we want to test
out a specific commit, branch, or tag of a module and test it in this new
environment? This is frequently what you'll be doing - making a change to
an existing module, pushing your change to a topic branch of that module's
repository, tying it to an environment (or creating a new environment by
branching the main Puppet repository), and then testing the change.

Let's go back to my 'notifyme' module that I've cloned to my laptop and push
a change to a BRANCH of that module's Github repository:

```
└(~/src/puppet-notifyme)▷ git branch
* master

└(~/src/puppet-notifyme)▷ git checkout -b change_the_message
Switched to a new branch 'change_the_message'

└(~/src/puppet-notifyme)▷ vim manifests/init.pp
## Make changes to the notify message

└(~/src/puppet-notifyme)▷ git add manifests/init.pp

└(~/src/puppet-notifyme)▷ git commit
[change_the_message bc3975b] Change the Message
 1 file changed, 1 insertion(+), 1 deletion(-)

└(~/src/puppet-notifyme)▷ git push origin change_the_message:change_the_message
Counting objects: 7, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (3/3), done.
Writing objects: 100% (4/4), 448 bytes, done.
Total 4 (delta 0), reused 0 (delta 0)
To https://github.com/glarizza/puppet-notifyme.git
 * [new branch]      change_the_message -> change_the_message

└(~/src/puppet-notifyme)▷ git branch -a
* change_the_message
  master
  remotes/origin/change_the_message
  remotes/origin/master

└(~/src/puppet-notifyme)▷ git log
commit bc3975bb5c75ada86bfc2c45db628b5a156f85ce
Author: Gary Larizza <gary@puppetlabs.com>
Date:   Wed Feb 19 13:55:26 2014 -0800

    Change the Message

    This commit changes the message to test my workflow.
```

What I'm showing you is the workflow that creates a new local branch called
'change_the_message' to the notifyme module, changes the message in my notify
resource, commits the change, and pushes the changes to a remote branch ALSO
called 'change_the_message'.

Because I created a topic branch, I can provide that branch name in the
`Puppetfile` located in the 'garysawesomeenvironment' branch of the 
[main Puppet repo][mainrepo]. THAT is the piece that ties together the specific
version of the module with the Puppet environment we want on the Puppet master.
Here's that change:

{% codeblock lang:ruby Puppetfile %}
mod "notifyme",
  :git => "git://github.com/glarizza/puppet-notifyme.git",
  :ref => 'change_the_message'
{% endcodeblock %}

Again, that change gets put into the 'garysawesomeenvironment' branch of the
[main Puppet repo][mainrepo] and pushed up to the remote:

```
└(~/src/puppet_repository)▷ vim Puppetfile
## Make changes

└(~/src/puppet_repository)▷ git add Puppetfile

└(~/src/puppet_repository)▷ git commit
[garysawesomeenvironment 89b139c] Update garysawesomeenvironment
 1 file changed, 2 insertions(+), 1 deletion(-)

└(~/src/puppet_repository)▷ git push origin garysawesomeenvironment:garysawesomeenvironment
Counting objects: 5, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 411 bytes, done.
Total 3 (delta 1), reused 0 (delta 0)
To https://github.com/glarizza/puppet_repository.git
   5239538..89b139c  garysawesomeenvironment -> garysawesomeenvironment

└(~/src/puppet_repository)▷ git log -p
commit 89b139c8c2faa888a402b98ea76e4ca138b3463d
Author: Gary Larizza <gary@puppetlabs.com>
Date:   Wed Feb 19 14:04:18 2014 -0800

    Update garysawesomeenvironment

    Tie this environment to the 'change_the_message' branch of my notifyme module.

diff --git a/Puppetfile b/Puppetfile
index 5e5d091..27fc06e 100644
--- a/Puppetfile
+++ b/Puppetfile
@@ -31,4 +31,5 @@ mod 'redis',
   :ref => 'feature/debian_support'

 mod "notifyme",
-  :git => "git://github.com/glarizza/puppet-notifyme.git"
+  :git => "git://github.com/glarizza/puppet-notifyme.git",
+  :ref => 'change_the_message'
```

Now let's synchronize again!!

```
[root@master1 garysawesomeenvironment]# `r10k deploy environment -pv`
[R10K::Task::Deployment::DeployEnvironments - INFO] Loading environments from all sources
<snip>
[R10K::Task::Environment::Deploy - NOTICE] Deploying environment garysawesomeenvironment
[R10K::Task::Puppetfile::Sync - INFO] Loading modules from Puppetfile into queue
<snip>
```

Cool, let's check our work on the VM:

```
[root@master1 garysawesomeenvironment]# pwd
/etc/puppetlabs/puppet/environments/garysawesomeenvironment
[root@master1 garysawesomeenvironment]# git branch
* garysawesomeenvironment
  master
```

And finally, let's run Puppet:

```
root@master1 garysawesomeenvironment]# puppet agent -t --environment garysawesomeenvironment
Info: Retrieving plugin
<snip fact loading>
Info: Caching catalog for master1
Info: Applying configuration version '1392847743'
Notice: This is the changed message in the change_the_message branch
Notice: /Stage[main]/Notifyme/Notify[This is the changed message in the change_the_message branch]/message: defined 'message' as 'This is the changed message in the change_the_message branch'
Notice: Finished catalog run in 12.10 seconds
```

TADA! We've successfully tied a specific version of a module to a specific
dynamic environment, deployed it to a master, and tested it out! Smell that?
That's the smell of awesome. Or Jeff in the next cubicle eating a burrito.
Either way, I like it.

### Merge your changes with master/production

It's green - fuck it; ship it! NOW you're speaking 'agile'! Assuming everything
went according to plan, let's merge our changes in with the production
environment and synchronize.  This is up to your company's workflow docs
(whether you use pull requests, a merge master, or poke Patrick and tell
him to tell Andy to merge in your change). I'm using git and Github, so
let's merge.

First, do the Module:

```
└(~/src/puppet-notifyme)▷ git checkout master
Switched to branch 'master'

└(~/src/puppet-notifyme)▷ git merge change_the_message
Updating d44a790..bc3975b
Fast-forward
 manifests/init.pp | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)

└(~/src/puppet-notifyme)▷ git push origin master:master
Total 0 (delta 0), reused 0 (delta 0)
To https://github.com/glarizza/puppet-notifyme.git
   d44a790..bc3975b  master -> master

└(~/src/puppet-notifyme)▷ cat manifests/init.pp
class notifyme {
  notify { "This is the changed message in the change_the_message branch": }
}
```

So now we have an issue, and that issue is that the production environment
has YET to have the 'notifyme' module added to it.  If we merge the contents
of the 'garysawesomeenvironment' branch with the 'production' branch of the
[main Puppet repo][mainrepo], then we're going to be pointing at the
'change_the_message' branch of the 'notifyme' module (because that was our last
commit).

Because of this, I can't do a straight merge, can I? For posterity's sake (in
the event that someone in the future wants to look for that branch on my Github
repo), I'm going to keep that branch alive. In a production environment, I
most likely would NOT have additional branches open for all my component modules
as that would get pretty annoying/confusing. Understand that this is a one-off
case because I'm doing a demo. BECAUSE of this, I'm going to modify the
`Puppetfile` in the 'production' branch of the [main Puppet repo][mainrepo]:

```
└(~/src/puppet_repository)▷ git checkout production
Switched to branch 'production'

└(~/src/puppet_repository)▷ vim Puppetfile
## Make changes here

└(~/src/puppet_repository)▷ git add Puppetfile

└(~/src/puppet_repository)▷ git commit
[production a74f269] Add notifyme module to Production environment
 1 file changed, 4 insertions(+)

└(~/src/puppet_repository)▷ git push origin production:production
Counting objects: 5, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 362 bytes, done.
Total 3 (delta 1), reused 0 (delta 0)
To https://github.com/glarizza/puppet_repository.git
   5ecefc8..a74f269  production -> production

└(~/src/puppet_repository)▷ git log -p
commit a74f26975102f3786eedddace89bda086162d801
Author: Gary Larizza <gary@puppetlabs.com>
Date:   Wed Feb 19 14:24:05 2014 -0800

    Add notifyme module to Production environment

diff --git a/Puppetfile b/Puppetfile
index 0b1da68..9168a81 100644
--- a/Puppetfile
+++ b/Puppetfile
@@ -29,3 +29,7 @@ mod "property_list_key",
 mod 'redis',
   :git => 'git://github.com/glarizza/puppet-redis',
   :ref => 'feature/debian_support'
+
+mod 'notifyme',
+  :git => 'git://github.com/glarizza/puppet-notifyme'
+
```

Alright, we've updated the production environment, now synchronize again
(I'll spare you and do it WITHOUT verbose mode):

```
[root@master1 garysawesomeenvironment]# r10k deploy environment -p
```

Okay, now run Puppet with the PRODUCTION environment:

```
[root@master1 garysawesomeenvironment]# puppet agent -t --environment production
Info: Retrieving plugin
<snipping fact loading>
Info: Caching catalog for master1
Info: Applying configuration version '1392848588'
Notice: This is the changed message in the change_the_message branch
Notice: /Stage[main]/Notifyme/Notify[This is the changed message in the change_the_message branch]/message: defined 'message' as 'This is the changed message in the change_the_message branch'
Notice: Finished catalog run in 12.66 seconds
```

Beautiful, we're synchronized!!!

### Making a change to an EXISTING module in an environment

Okay, so we saw previously how to add a NEW module to an environment, but what
if we already HAVE a module in an environment and we want to make an update/change
to it?  Well, it's largely the same process:

* Cut a branch to the module
* Commit your code and push it up to the module's repo
* Cut a branch to the [main Puppet repo][mainrepo]
* Push that branch up to the [main Puppet repo][mainrepo]
* Perform an R10k synchronization to sync the environments
* Test your changes
* Merge the changes with the master branch of the module
* DONE!

Let's go back and change that notify message again, shall we?

```
└(~/src/puppet-notifyme)▷ git checkout -b 'another_change'
Switched to a new branch 'another_change'

└(~/src/puppet-notifyme)▷ vim manifests/init.pp
## Make changes to the message

└(~/src/puppet-notifyme)▷ git add manifests/init.pp

└(~/src/puppet-notifyme)▷ git commit
[another_change 608166e] Change the message that already exists!
 1 file changed, 1 insertion(+), 1 deletion(-)

└(~/src/puppet-notifyme)▷ git push origin another_change:another_change
Counting objects: 7, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (3/3), done.
Writing objects: 100% (4/4), 426 bytes, done.
Total 4 (delta 0), reused 0 (delta 0)
To https://github.com/glarizza/puppet-notifyme.git
 * [new branch]      another_change -> another_change
```

Okay, let's re-use 'garysawesomeenvironment' because I like the name, but
tie it to the new 'another_change' branch of the 'notifyme' module:

```
└(~/src/puppet_repository)▷ git checkout garysawesomeenvironment
Switched to branch 'garysawesomeenvironment'

└(~/src/puppet_repository)▷ vim Puppetfile
## Make change to Puppetfile to tie it to 'another_change' branch

└(~/src/puppet_repository)▷ git add Puppetfile

└(~/src/puppet_repository)▷ git commit
[garysawesomeenvironment ce84a30] Tie garysawesomeenvironment to 'another_change'
 1 file changed, 1 insertion(+), 1 deletion(-)

└(~/src/puppet_repository)▷ git push origin garysawesomeenvironment:garysawesomeenvironment
Counting objects: 5, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 386 bytes, done.
Total 3 (delta 1), reused 0 (delta 0)
To https://github.com/glarizza/puppet_repository.git
   89b139c..ce84a30  garysawesomeenvironment -> garysawesomeenvironment
```

The `Puppetfile` for that branch now has an entry for the 'notifyme' module
that looks like this:

{% codeblock lang:ruby Puppetfile %}
mod "notifyme",
  :git => "git://github.com/glarizza/puppet-notifyme.git",
  :ref => 'another_change'
{% endcodeblock %}

Okay, synchronize again!

```
[root@master1 garysawesomeenvironment]# r10k deploy environment -p
```

And now run Puppet in the 'garysawesomeenvironment' environment:

```
[root@master1 garysawesomeenvironment]# puppet agent -t --environment garysawesomeenvironment
Info: Retrieving plugin
<snip fact loading>
Info: Caching catalog for master1
Info: Applying configuration version '1392849521'
Notice: This changes the message that already exists!!!!
Notice: /Stage[main]/Notifyme/Notify[This changes the message that already exists!!!!]/message: defined 'message' as 'This changes the message that already exists!!!!'
Notice: Finished catalog run in 12.54 seconds
```

There's the message that I changed in the 'another_change' branch of my 'notifyme'
module!  What's it look like if I run in the 'production' environment, though?

```

ot@master1 garysawesomeenvironment]# puppet agent -t --environment production
Info: Retrieving plugin
<snip fact loading>
Info: Caching catalog for master1
Info: Applying configuration version '1392848588'
Notice: This is the changed message in the change_the_message branch
Notice: /Stage[main]/Notifyme/Notify[This is the changed message in the change_the_message branch]/message: defined 'message' as 'This is the changed message in the change_the_message branch'
Notice: Finished catalog run in 14.11 seconds
```

There's the old message that's in the 'master' branch of the 'notifyme'
module (which is where the 'production' branch `Puppetfile` is pointing).
To merge the changes into the production environment, we now only have to
do one thing: that's merge the changes in the 'another_change' branch of
the 'notifyme' module to the 'master' branch - that's it!  Why? Because
the `Puppetfile` in the `production` branch of the [main Puppet repo][mainrepo]
(and thus the production Puppet ENVIRONMENT) is already POINTING at the
master branch of the 'notifyme' module.  Let's do the merge:

```
└(~/src/puppet-notifyme)▷ git checkout master
Switched to branch 'master'

└(~/src/puppet-notifyme)▷ git merge another_change
Updating bc3975b..608166e
Fast-forward
 manifests/init.pp | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)

└(~/src/puppet-notifyme)▷ git push origin master:master
Total 0 (delta 0), reused 0 (delta 0)
To https://github.com/glarizza/puppet-notifyme.git
   bc3975b..608166e  master -> master
```

Another R10k synchronization is needed on the master:

```
[root@master1 garysawesomeenvironment]# r10k deploy environment -p
```

And now let's run Puppet in the production environment:

```
[root@master1 garysawesomeenvironment]# puppet agent -t --environment production
Info: Retrieving plugin
<snip fact loading>
Info: Caching catalog for master1
Info: Applying configuration version '1392850004'
Notice: This changes the message that already exists!!!!
Notice: /Stage[main]/Notifyme/Notify[This changes the message that already exists!!!!]/message: defined 'message' as 'This changes the message that already exists!!!!'
Notice: Finished catalog run in 11.82 seconds
```

There's the message that was previously in the 'another_change' branch that's
been merged to the 'master' branch (and thus is entered into the production
Puppet environment).

## Holy crap, that's a lot to take in...

Yeah, tell me about it. And, believe it or not, I'm STILL not done with
everything that I want to talk about regarding R10k - there's still more
info on:

* Incorporating Hiera data
* Triggering R10k with MCollective
* Tying R10k to CI workflow

Those will come in a later post once I have time to decide how to tackle them.
Until then, this should give you more than enough information to get started
with R10k in your own environment.

If you have any questions/comments/corrections, PLEASE enter them in the
comments below and I'll be happy to respond when I'm not flying from gig to
gig! :)  Cheers!

[firstpost]: http://garylarizza.com/blog/2014/02/17/puppet-workflow-part-1/
[secondpost]: http://garylarizza.com/blog/2014/02/17/puppet-workflow-part-2/
[puppetenv]: http://docs.puppetlabs.com/guides/environment.html
[envbug]: http://projects.puppetlabs.com/issues/12173
[puppetgit]: http://bit.ly/puppetgit
[librarian]: https://github.com/rodjek/librarian-puppet
[finchlibrarian]: http://somethingsinistral.net/blog/scaling-puppet-environment-deployment/
[finchr10k]: http://somethingsinistral.net/blog/scaling-puppet-environment-deployment/
[mainrepo]: https://github.com/glarizza/puppet_repository
[forgemod]: http://forge.puppetlabs.com/zack/r10k
[zack]: https://github.com/acidprime
[notifyme]: https://github.com/glarizza/puppet-notifyme
