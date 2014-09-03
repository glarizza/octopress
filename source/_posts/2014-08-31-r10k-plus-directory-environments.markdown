---
layout: post
title: "R10k + Directory Environments"
date: 2014-08-31 12:00:00 -0700
comments: true
categories: ['puppet', 'r10k', 'workflow']
---

If you've read anything I've posted in the past year, you know my feelings about
the word 'environments' and about how well we tend to name things here at
Puppet Labs (and if you don't, [you can check out that post here][env_blog]).
Since then, Puppet Labs has released a new feature called [directory
environments (click this link for further reading)][directory_environments]
that replace the older 'config file environments' that we all used to use (i.e.
stanzas in puppet.conf).  Directory environments weren't without their false
starts and issues, but further releases of Puppet, and their inclusion in
Puppet Enterprise 3.3.0, have allowed more people
to ask about them.  SO, I thought I'd do a quick writeup about them...

## R10k had a child: Directory Environments

The Puppet platform team had a couple of problems with config file environments
in puppet.conf - namely:

* Entering them in puppet.conf meant that you couldn't use environments named 'master', 'main', or 'agent'
* There was no easy/reliable way to determine all the available/used Puppet environments without making assumptions (and hacky code) - especially if someone were using R10k + dynamic environments
* Adding more environments to `puppet.conf` made managing that file something of a nightmare (`environments.d` anyone?)

Combine this with the fact that [most of the Professional Services team was
rolling out R10k to create dynamic environments][r10k_post] (which meant we
were abusing `$environment` inside `puppet.conf` and creating environments...well...
dynamically and on-the-fly), and they knew something needed to be done.
Because R10k was so popular and widely deployed, an environment solution that
was a simple step-up from an R10k deployment was made the target, and directory
environments were born.

## How does it work?

Directory environments, essentially, are born out of a folder on the Puppet master
(typically `$confdir/environments`, where `$confdir` is `/etc/puppetlabs/puppet`
in Puppet Enterprise) wherein every subfolder is a new Puppet environment. Every
subfolder contains a couple of key items:

* A `modules` folder containing all modules for that environment
* A `manifests/site.pp` file containing the site.pp file for that environment
* A new `environment.conf` file which can be used to set the `modulepath`, the `environment_timeout`, and, a new and often-requested feature, the ability to have environment-specific `config_version` settings

Basically, it's everything that R10k ALREADY does with a couple of added goodies
dropped into an `environment.conf` file. [Feel free to read the official docs
on configuring directory environments][configure_dir_env] for further information
on all of the goodies!

## Cool, how do we set it up?

It wouldn't be one of my blog posts if it didn't include exact steps to configure
shit, would it? For this walkthrough, I'm using a Centos 6.5 vm with DNS working
(i.e. the node can ping itself and knows its own hostname and FQDN), and I've
already installed an All-in-one installation of Puppet Enterprise 3.3.0. For
the walkthrough, we're going to setup:

* Directory environments based on a control repo
* Hiera data inside a `hieradata` folder in the control repo
* Hiera to use the per-environment hieradata folder

Let's start to break down the components:

### The 'Control Repo'?

Sometime between [my initial R10k post][r10k_post] and THIS post, the Puppet Labs PS
team has come to call the repository that contains the Puppetfile and is used
to track Puppet environments on all Puppet masters the 'Control Repo' (because
it 'Controls the creation of Puppet environments', ya dig?  Zack Smith and
James Sweeny are actually pretty tickled about making that name stick). For
the purpose of this demonstration, I'm using a repository on Github:

[https://github.com/glarizza/puppet_repository](https://github.com/glarizza/puppet_repository)

Everything you will need for this walkthrough is in that repository, and we
will refer to it frequently. You DO NOT need to use my repository, and it's
definitely going to be required that you create your OWN, but it's there
for reference purposes (and to give you a couple of Puppet manifests to
make setup a bit easier).

### Configuring the Puppet master

We're going to first clone my control repo to `/tmp` so we can use it to
configure R10k and the Puppet master itself:

```
[root@master ~]# cd /tmp

[root@master /tmp]# git clone https://github.com/glarizza/puppet_repository.git
Initialized empty Git repository in /tmp/puppet_repository/.git/
remote: Counting objects: 164, done.
remote: Compressing objects: 100% (134/134), done.
remote: Total 164 (delta 54), reused 81 (delta 16)
Receiving objects: 100% (164/164), 22.68 KiB, done.
Resolving deltas: 100% (54/54), done.

[root@master /tmp]# cd puppet_repository
```

Great, I've cloned my repo. To configure R10k, we're going to need to pull
down Zack Smith's R10k module from the forge with `puppet module install zack/r10k`
and then use `puppet apply` on a manifest in my repo with
`puppet apply configure_r10k.pp`.  **DO NOTE: If you want to use YOUR Control
Repo, and NOT the one I use on Github, then you need to modify the
`configure_r10k.pp` file and replace the `remote` property with the URL to
YOUR Control Repo that's housed on a git repository!**

```
[root@master /tmp/puppet_repository:production]# puppet module install zack/r10k

Notice: Preparing to install into /etc/puppetlabs/puppet/modules ...
Notice: Downloading from https://forgeapi.puppetlabs.com ...
Notice: Found at least one version of puppetlabs-stdlib compatible with PE (3.3.0);
Notice: Skipping versions which don't express PE compatibility. To install
the most recent version of the module regardless of compatibility
with PE, use the '--ignore-requirements' flag.
Notice: Found at least one version of puppetlabs-inifile compatible with PE (3.3.0);
Notice: Skipping versions which don't express PE compatibility. To install
the most recent version of the module regardless of compatibility
with PE, use the '--ignore-requirements' flag.
Notice: Found at least one version of puppetlabs-vcsrepo compatible with PE (3.3.0);
Notice: Skipping versions which don't express PE compatibility. To install
the most recent version of the module regardless of compatibility
with PE, use the '--ignore-requirements' flag.
Notice: Found at least one version of puppetlabs-concat compatible with PE (3.3.0);
Notice: Skipping versions which don't express PE compatibility. To install
the most recent version of the module regardless of compatibility
with PE, use the '--ignore-requirements' flag.
Notice: Installing -- do not interrupt ...
/etc/puppetlabs/puppet/modules
└─┬ zack-r10k (v2.2.7)
  ├─┬ gentoo-portage (v2.2.0)
  │ └── puppetlabs-concat (v1.0.3) [/opt/puppet/share/puppet/modules]
  ├── mhuffnagle-make (v0.0.2)
  ├── puppetlabs-gcc (v0.2.0)
  ├── puppetlabs-git (v0.2.0)
  ├── puppetlabs-inifile (v1.1.0) [/opt/puppet/share/puppet/modules]
  ├── puppetlabs-pe_gem (v0.0.1)
  ├── puppetlabs-ruby (v0.2.1)
  ├── puppetlabs-stdlib (v3.2.2) [/opt/puppet/share/puppet/modules]
  └── puppetlabs-vcsrepo (v1.1.0)

[root@master /tmp/puppet_repository:production]# puppet apply configure_r10k.pp

Notice: Compiled catalog for master.puppetlabs.vm in environment production in 0.71 seconds
Warning: The package type's allow_virtual parameter will be changing its default value from false to true in a future release. If you do not want to allow virtual packages, please explicitly set allow_virtual to false.
   (at /opt/puppet/lib/ruby/site_ruby/1.9.1/puppet/type.rb:816:in `set_default')
Notice: /Stage[main]/R10k::Install/Package[r10k]/ensure: created
Notice: /Stage[main]/R10k::Install::Pe_gem/File[/usr/bin/r10k]/ensure: created
Notice: /Stage[main]/R10k::Config/File[r10k.yaml]/ensure: defined content as '{md5}5cda58e8a01e7ff12544d30105d13a2a'
Notice: Finished catalog run in 11.24 seconds
```

Performing those commands will successfully setup R10k to point to my Control
Repo out on Github (and, again, if you don't WANT that, then you need to make
the change to the `remote` property in `configure_r10k.pp`). We next need to
configure Directory Environments in `puppet.conf` by setting two attributes:

* `environmentpath` (Or the path to the folder containing environments)
* `basemodulepath` (Or, the set of modules that will be shared across ALL ENVIRONMENTS)

I have created a Puppet manifest that will set these attributes, and this
manifest requires the `puppetlabs/inifile` module from the Puppet Forge.
Fortunately, since I'm using Puppet Enterprise, that module is already installed.
If you're using open source Puppet and the module is NOT installed, feel free
to install it by running `puppet module install puppetlabs/inifile`. Once
this is done, go ahead and execute the manifest by running
`puppet apply configure_directory_environments.pp`:

```
[root@master /tmp/puppet_repository:production]# puppet apply configure_directory_environments.pp

Notice: Compiled catalog for master.puppetlabs.vm in environment production in 0.05 seconds
Notice: /Stage[main]/Main/Ini_setting[Configure environmentpath]/ensure: created
Notice: /Stage[main]/Main/Ini_setting[Configure basemodulepath]/value: value changed '/etc/puppetlabs/puppet/modules:/opt/puppet/share/puppet/modules' to '$confdir/modules:/opt/puppet/share/puppet/modules'
Notice: Finished catalog run in 0.20 seconds
```

The last step to configuring the Puppet master is to execute an R10k run.
We can do that by running `r10k deploy environment -pv`:

```
[root@master /tmp/puppet_repository:production]# r10k deploy environment -pv

[R10K::Source::Git - INFO] Determining current branches for "https://github.com/glarizza/puppet_repository.git"
[R10K::Task::Environment::Deploy - NOTICE] Deploying environment production
[R10K::Task::Puppetfile::Sync - INFO] Loading modules from Puppetfile into queue
[R10K::Task::Module::Sync - INFO] Deploying profiles into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying notifyme into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying ntp into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying apache into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying profiles into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying notifyme into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying ntp into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Module::Sync - INFO] Deploying apache into /etc/puppetlabs/puppet/environments/production/modules
[R10K::Task::Environment::Deploy - NOTICE] Deploying environment webinar_env
[R10K::Task::Puppetfile::Sync - INFO] Loading modules from Puppetfile into queue
[R10K::Task::Module::Sync - INFO] Deploying profiles into /etc/puppetlabs/puppet/environments/webinar_env/modules
[R10K::Task::Module::Sync - INFO] Deploying notifyme into /etc/puppetlabs/puppet/environments/webinar_env/modules
[R10K::Task::Module::Sync - INFO] Deploying haproxy into /etc/puppetlabs/puppet/environments/webinar_env/modules
[R10K::Task::Module::Sync - INFO] Deploying mysql into /etc/puppetlabs/puppet/environments/webinar_env/modules
[R10K::Task::Module::Sync - INFO] Deploying ntp into /etc/puppetlabs/puppet/environments/webinar_env/modules
[R10K::Task::Module::Sync - INFO] Deploying apache into /etc/puppetlabs/puppet/environments/webinar_env/modules
[R10K::Task::Module::Sync - INFO] Deploying profiles into /etc/puppetlabs/puppet/environments/webinar_env/modules
[R10K::Task::Module::Sync - INFO] Deploying notifyme into /etc/puppetlabs/puppet/environments/webinar_env/modules
[R10K::Task::Module::Sync - INFO] Deploying haproxy into /etc/puppetlabs/puppet/environments/webinar_env/modules
[R10K::Task::Module::Sync - INFO] Deploying mysql into /etc/puppetlabs/puppet/environments/webinar_env/modules
[R10K::Task::Module::Sync - INFO] Deploying ntp into /etc/puppetlabs/puppet/environments/webinar_env/modules
[R10K::Task::Module::Sync - INFO] Deploying apache into /etc/puppetlabs/puppet/environments/webinar_env/modules
[R10K::Task::Deployment::PurgeEnvironments - INFO] Purging stale environments from /etc/puppetlabs/puppet/environments
```

Great!  Everything should be setup (if you're using my repo)!  My repository has
a production branch, which is what Puppet's default environment is named,
so we can test that everything works by listing out all modules in the main
production environment with the `puppet module list` command:

```
[root@master /tmp/puppet_repository:production]# puppet module list

Warning: Module 'puppetlabs-stdlib' (v3.2.2) fails to meet some dependencies:
  'puppetlabs-ntp' (v3.1.2) requires 'puppetlabs-stdlib' (>= 4.0.0)
/etc/puppetlabs/puppet/environments/production/modules
├── notifyme (???)
├── profiles (???)
├── puppetlabs-apache (v1.1.1)
└── puppetlabs-ntp (v3.1.2)
/etc/puppetlabs/puppet/modules
├── gentoo-portage (v2.2.0)
├── mhuffnagle-make (v0.0.2)
├── puppetlabs-gcc (v0.2.0)
├── puppetlabs-git (v0.2.0)
├── puppetlabs-pe_gem (v0.0.1)
├── puppetlabs-ruby (v0.2.1)
├── puppetlabs-vcsrepo (v1.1.0)
└── zack-r10k (v2.2.7)
/opt/puppet/share/puppet/modules
├── puppetlabs-apt (v1.5.0)
├── puppetlabs-auth_conf (v0.2.2)
├── puppetlabs-concat (v1.0.3)
├── puppetlabs-firewall (v1.1.2)
├── puppetlabs-inifile (v1.1.0)
├── puppetlabs-java_ks (v1.2.4)
├── puppetlabs-pe_accounts (v2.0.2-3-ge71b5a0)
├── puppetlabs-pe_console_prune (v0.1.1-4-g293f45b)
├── puppetlabs-pe_mcollective (v0.2.10-15-gb8343bb)
├── puppetlabs-pe_postgresql (v1.0.4-4-g0bcffae)
├── puppetlabs-pe_puppetdb (v1.1.1-7-g8cb11bf)
├── puppetlabs-pe_razor (v0.2.1-1-g80acb4d)
├── puppetlabs-pe_repo (v0.7.7-32-gfd1c97f)
├── puppetlabs-pe_staging (v0.3.3-2-g3ed56f8)
├── puppetlabs-postgresql (v2.5.0-pe2)
├── puppetlabs-puppet_enterprise (v3.2.1-27-g8f61956)
├── puppetlabs-reboot (v0.1.4)
├── puppetlabs-request_manager (v0.1.1)
└── puppetlabs-stdlib (v3.2.2)  invalid
```

Notice a couple of things:

* First, I've got some dependency issues...oh well, nothing that's a game-stopper
* Second, the path to the production environment's module is correct at: `/etc/puppetlabs/puppet/environments/production/modules`

### Configuring Hiera

The last dinghy to be configured on this dreamboat is Hiera. Hiera is Puppet's
data lookup mechanism, and is used to gather specific bits of data (such
as versions of packages, hostnames, passwords, and other business-specific
data). Explaining HOW Hiera works is beyond the scope of this article, but
configuring Hiera data on a per-environment basis IS absolutely a worthwhile
endeavor.

In this example, I'm going to demonstrate coupling Hiera data with the Control
Repo for simple replication of Hiera data across environments. You COULD also
choose to put your Hiera data in a separate repository and set it up in
`/etc/r10k.yaml` as another source, but that exercise is left to the reader
[(and if you're interested, I talk about it in this post).][3bpost]

You'll notice that my demonstration repository ALREADY includes Hiera data,
and so that data is automatically being replicated to all environments. By
default, Hiera's configuration file (`hiera.yaml`) has no YAML data directory
specified, so we'll need to make that change.  [In my demonstration control
repository, I've included a sample `hiera.yaml`,][hieradotyaml] but let's take a look at
one below:

```yaml
## /etc/puppetlabs/puppet/hiera.yaml

---
:backends:
  - yaml
:hierarchy:
  - "%{clientcert}"
  - "%{application_tier}"
  - common

:yaml:
# datadir is empty here, so hiera uses its defaults:
# - /var/lib/hiera on *nix
# - %CommonAppData%\PuppetLabs\hiera\var on Windows
# When specifying a datadir, make sure the directory exists.
  :datadir: "/etc/puppetlabs/puppet/environments/%{environment}/hieradata"
```

This hiera.yaml file specifies a hierarchy with three levels - a node-specific,
level, a level for different application tiers (like 'dev', 'test', 'prod', and
etc), and finally makes the change we need: mapping the data directory to each
environment's hieradata folder.  The path to `hiera.yaml` is Puppet's
configuration directory (which is `/etc/puppetlabs/puppet` for Puppet
Enterprise, or `/etc/puppet` for the open source version of Puppet), so open
the file there, make your changes, and finally you'll need to need to restart
the Puppet master service to have the changes picked up.

Next, let's perform a test by executing the `hiera` binary from the command
line before running puppet:

```
[root@master /etc/puppetlabs/puppet/environments]# hiera message environment=production
This node is using common data

[root@master /etc/puppetlabs/puppet/environments]# hiera message environment=webinar_env -d
DEBUG: 2014-08-31 19:55:44 +0000: Hiera YAML backend starting
DEBUG: 2014-08-31 19:55:44 +0000: Looking up message in YAML backend
DEBUG: 2014-08-31 19:55:44 +0000: Looking for data source common
DEBUG: 2014-08-31 19:55:44 +0000: Found message in common
This node is using common data

[root@master /etc/puppetlabs/puppet/environments]# hiera message environment=bad_env -d
DEBUG: 2014-08-31 19:58:22 +0000: Hiera YAML backend starting
DEBUG: 2014-08-31 19:58:22 +0000: Looking up message in YAML backend
DEBUG: 2014-08-31 19:58:22 +0000: Looking for data source common
DEBUG: 2014-08-31 19:58:22 +0000: Cannot find datafile /etc/puppetlabs/puppet/environments/bad_env/hieradata/common.yaml, skipping
nil
```

You can see that for the first example, I passed the environment of `production`
and did a simple lookup for a key called `message` - Hiera then returned me
the value of out that environment's `common.yaml` file.  Next, I did another
lookup, but added `-d` to enable debug mode (debug mode on the `hiera`
binary is REALLY handy for debugging problems with Hiera - combine it with
specifying values from the command line, and you can pretty quickly simulate
what value a node is going to get).  Notice the last example where I specified
an invalid environment - Hiera logged that it couldn't find the datafile
requested and ultimately returned a nil, or empty, value.

Since we're working on the Puppet master machine, we can even check for a value
using `puppet apply` combined with the notice function:

```
[root@master /etc/puppetlabs/puppet/environments]# puppet apply -e "notice(hiera('message'))"
Notice: Scope(Class[main]): This node is using common data
Notice: Compiled catalog for master.puppetlabs.vm in environment production in 0.09 seconds
Notice: Finished catalog run in 0.19 seconds
```

Great, it's working, but let's look at pulling data from a higher level in the
hierarchy - like from the `application_tier` level. We haven't defined an
`application_tier` fact, however, so we'll need to fake it. First, let's do
that with the `hiera` binary:

```
[root@master /etc/puppetlabs/puppet/environments]# hiera message environment=production application_tier=dev -d
DEBUG: 2014-08-31 20:04:12 +0000: Hiera YAML backend starting
DEBUG: 2014-08-31 20:04:12 +0000: Looking up message in YAML backend
DEBUG: 2014-08-31 20:04:12 +0000: Looking for data source dev
DEBUG: 2014-08-31 20:04:12 +0000: Found message in dev
You are in the development application tier
```

And then also with `puppet apply`:

```
[root@master /etc/puppetlabs/puppet/environments]# FACTER_application_tier=dev puppet apply -e "notice(hiera('message'))"
Notice: Scope(Class[main]): You are in the development application tier
Notice: Compiled catalog for master.puppetlabs.vm in environment production in 0.09 seconds
Notice: Finished catalog run in 0.18 seconds
```

## Tuning `environment.conf`

The brand-new, per-environment  `environment.conf` file is meant to be (for
the most part) a one-stop-shop for your Puppet environment tuning needs. Right
now, the only things you'll need to tune will be the `modulepath`,
`config_version`, and possibly the `environment_timeout`.

### Module path

Before directory environments, every environment had its own `modulepath` that
needed to be tuned to allow for modules that were to be used by this
machine/environment, as well as shared modules.  That `modulepath` worked like
`$PATH` in that it was a priority-based lookup for modules (i.e. the first
directory in `modulepath` that had a module matching the module name you wanted
won).  It also previously required the FULL path to be used for every path in
`modulepath`.

Those days are over.

As I mentioned before, the main `puppet.conf` configuration file has a new
parameter called `basemodulepath` that can be used to specify modules that are
to be shared across ALL modules in ALL environments. Paths defined here
(typically `$confdir/modules` and `/opt/puppet/share/puppet/modules`) are
usually put at the END of a `modulepath` so Puppet can search for any
overridden modules that show up in earlier `modulepath` paths. In the previous
configuration steps, we executed a manifest that setup `basemodulepath` to
look like:

```
basemodulepath = $confdir/modules:/opt/puppet/share/puppet/modules
```

Again, feel free to add or remove paths (except don't remove
`/opt/puppet/share/puppet/modules` if you're using Puppet Enterprise, because
that's where all Puppet Enterprise modules are located), especially if you're
using a giant monolithic repo of modules (which was typically done before things
like R10k evolved).

With `basemodulepath` configured, it's now time to configure the `modulepath`
to be defined for every environment. My demonstration control repo contains
a sample `environment.conf` that defines a `modulepath` like so:

```
modulepath = modules:$basemodulepath
```

You'll notice, now, that there are relative paths in `modulepath`. This is
possible because now each environment contains an `environment.conf`, and thus
relative paths make sense. In this example, nodes in the production environment
(`/etc/puppetlabs/puppet/environments/production`) will look for a module by its
name FIRST by looking in a folder called `modules` inside the current
environment folder (i.e. `/etc/puppetlabs/puppet/environments/production/modules/<module_name>`).
If the module wasn't found there, it looks for the module in the order that
paths are defined for `basemodulepath` above. If Puppet fails to find a module
in ANY of the paths, a compile error is raised.

### Per-environment `config_version`

[Setting `config_version` has been around for awhile][config_version] - hell,
I remember video of Jeff McCune talking about it at the first Puppetcamp Europe
in like 2010 - but the new directory environments implementation has fine
tuned it a bit. Previously, `config_version` was a command executed on the
Puppet master at compile time to determine a string used for versioning the
configuration enforced during that Puppet run. When it's not set it defaults
to something of a time/date stamp off the parser, but it's way more useful to
make it do something like determine the most recent commit hash from a repository.

In the past when we used a giant monolithic repository containing all Puppet
modules, it was SUPER easy to get a single commit hash and be done. As everyone
moved their modules into individual repositories, determining *WHAT* you were
enforcing became harder. With the birth of R10k an the control repo, we
suddenly had something we could query for the state of our modules being
enforced. The problem existed, though, that with multiple dynamic environments
using multiple git branches, `config_version` wasn't easily tuned to be able
to grab the most recent commit from every branch.

Now that `config_version` is set in a per-environment `environment.conf`, we
can make `config_version` much smarter. Again, looking in the `environment.conf`
defined in my demonstration control repo produces this:

```
config_version = '/usr/bin/git --git-dir $confdir/environments/$environment/.git rev-parse HEAD'
```

This setting will cause the Puppet master to produce the most recent commit ID
for whatever environment you're in and embed it in the catalog and the report
that is sent back to the Puppet master after a Puppet run.

[I actually discovered a bug in `config_version` while writing this post][cv_bug],
and it's that `config_version` is subject to the relative pathing fun that other
`environment.conf` settings are subject to. Relative pathing is great for things like
`modulepath`, and it's even good for `config_version` if you're including the
script you want to run to gather the `config_version` string inside the control
repo, but using a one-line command that tries to execute a binary on the system
that DOESN'T include the full path to the binary causes an error (because Puppet
attempts to look for that binary in the current environment path, and NOT by
searching `$PATH` on the system).  Feel free to follow or comment on the bug
if the mood hits you.

### Caching and environment_timeout

The Puppet master loads environments on-request, but it also caches data associated
with each environment to make things faster. This caching is finally tunable on a
per-environment basis by defining the `environment_timeout` setting in
`environment.conf`.  The default setting is 3 minutes, which means the Puppet master
will invalidate its caches and reload environment data every 3 minutes, but that's
now tunable. [Definitely read up on this setting before making changes.][env_timeout]

## Classification

One of the last new features of directory environments is the ability to include
an environment-specific `site.pp` file for classification. You could ALWAYS do
this by modifying the `manifest` configuration item in `puppet.conf`, but now
each environment can have its own `manifest` setting. The default behavior is
to have the Puppet master look for `manifests/site.pp` in every environment
directory, and I really wouldn't change that unless you have a good reason. DO
NOTE, however, that if you're using Puppet Enterprise, you'll need to be careful
with your `site.pp` file.  Puppet Enterprise defines things like the Filebucket
and overrides for the File resource in `site.pp`, so if you're using Puppet Enterprise,
you'll need to copy those changes into the `site.pp` file you add into your control
repo (as I did).

It may take you a couple of times to change your thinking from looking at the main
`site.pp` in `$confdir/manifests` to looking at each environment-specific `site.pp`
file, but definitely take advantage of Puppet's commandline tool to help you track
which `site.pp` Puppet is monitoring:

```
[root@master /etc/puppetlabs/puppet/environments]# puppet config print manifest
/etc/puppetlabs/puppet/environments/production/manifests

[root@master /etc/puppetlabs/puppet/environments]# puppet config print manifest --environment webinar_env
/etc/puppetlabs/puppet/environments/webinar_env/manifests
```

You can see that `puppet config print` can be used to get the path to the
directory that contains `site.pp`.  Even cooler is what happens when you
specify an environment that doesn't exist:

```
[root@master /etc/puppetlabs/puppet/environments]# puppet config print manifest --environment bad_env
no_manifest
```

Yep, Puppet tells you if it can't find the manifest file.  That's pretty cool.


## Wrapping Up

Even though the new implementation of directory environments is meant to map
closely to a workflow most of us have been using (if you've been using R10k, that is),
there are still some new features that may take you by surprise. Hopefully this
post gets you started with just enough information to setup your own test
environment and start playing. PLEASE DO make sure to file bugs on any behavior
that comes as unexpected or stops you from using your existing workflow. Cheers!

[env_blog]: http://garylarizza.com/blog/2014/03/26/random-r10k-workflow-ideas/
[directory_environments]: https://docs.puppetlabs.com/puppet/latest/reference/environments.html
[r10k_post]: http://bit.ly/puppetworkflows3
[3bpost]: http://bit.ly/puppetworkflows3b
[configure_dir_env]: https://docs.puppetlabs.com/puppet/latest/reference/environments_configuring.html
[hieradotyaml]: https://github.com/glarizza/puppet_repository/blob/production/hiera.yaml
[config_version]: https://docs.puppetlabs.com/references/stable/configuration.html#configversion
[cv_bug]: https://tickets.puppetlabs.com/browse/PUP-3150
[env_timeout]: https://docs.puppetlabs.com/puppet/3.6/reference/environments_configuring.html#environmenttimeout
