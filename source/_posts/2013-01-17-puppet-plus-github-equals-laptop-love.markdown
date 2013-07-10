---
layout: post
title: "Puppet + Github = laptop <3"
date: 2013-02-15 
comments: true
categories: 
---

Everybody wants to be special (I blame our moms). The price of special, when
it comes to IT, is time. Consider how long you've spent on just your damn
PROMPT and you'll realize why automation gives any good sysadmin a case of
the giggles. Like the cobbler's kids, though, your laptop and development
environment are always the last to be visited by the automation gnomes.

Until now.

{% youtube YlKXRdSAZhY %}

[Will Farrington gave a great talk at Puppetconf 2012](http://www.youtube.com/watch?v=YlKXRdSAZhY)
about managing an army of developer laptops using Puppet + some Github love
that left more than a couple of people asking for his code. That request
has unfortunately been denied.

Until now.

## Boxen, and hand-tune no more

[Enter Boxen][boxen] (née 'The Setup'), a full-fledged open source project from
the guys at Github that melds Puppet with your Github credentials to create a
framework for automating everything from applications, to dotfiles, and even
printers and emacs extensions (that last bit's a lie - no one should be using
emacs).

How does it work? Think 'Masterless Puppet' (or, just a bunch of Puppet modules
that are enforced by running `puppet apply` on your local machine). Boxen not
only includes the framework ITSELF, but is a project on Github that hosts
over 75 individual modules for managing things like rbenv, homebrew, git, mysql,
postgres, riak, redis, npm, erlang, dropbox, skype, minecraft, heroku,
1password, iterm2, and much more. Odds are there's a module for many of the
common things you setup on your laptop. And what about things like dotfiles
that are intrinsically personal? You can create your own repository and manage
them like you would any other file/directory/symlink on the file system. The
goal is to model every little piece of your laptop that makes it 'unique' from
everyone else until you have your entire environment managed and the hardware
becomes...well...disposable. How many times have you shied away from doing an
upgrade because some component of your laptop required you to spend countless
hours tinkering with it? If you've done the time, you should do something to
make sure that you NEVER have to repeat that process manually ever again.

Boxen is also very self-contained. Packages and binaries that come out of
Homebrew are installed into `/opt/boxen/homebrew/bin`, frameworks like Ruby and
Node are installed into `/opt/boxen/{rbenv,nvm}`, and individual versions of
those frameworks are kept separate from your system version of those
frameworks.  These details are important when you consider that you could purge
the whole setup without having to rip out components scattered around your
system.

You may be reading this and thinking "There's no way in hell I can use this to
manage every laptop in my organization!", and you're right. The POINT of Boxen
is that it's a tool written by developers for developers to automate THEIR
systems. The goal of developing is to have as little friction between the
process of writing code and deploying that code into production. A tool like
Boxen allows you to more quickly GET to the state where your laptop is ready
for you to start developing. If you want a tool to completely manage and lock
down all the laptops on your system, look to using [Puppet](http://www.puppetlabs.com)
in agent/master mode or to a tool like [Munki](https://code.google.com/p/munki/)
to manage all packages on your system. If you're interested in giving your
developers/users the freedom to manage their OWN 'boxes' because they know best
what works for them, then Boxen is your tool.

There IS one catch - it's targeted to OS X (10.8 to be exact).


## Diary of an elitist

I was fortunate to have early-access to Boxen in order to kick its Ruby tyres.
As someone who's managed Macs with Puppet before (all the way down to the
desktop/laptop level), I was embarrassed to admit that I had NOTHING about
my laptop automated. [Will][wf] unlocked the project and basically said "Have
fun, break shit, fix it, and file pull requests" and away I went. To commit
completely to the project, I did what any sane person would do.

I reformatted my laptop and started entirely from scratch.

(Let's be clear here - you don't have to do that. Initially Will reported
problems getting Boxen running in VMs, but I never ran into an issue. I ran
Boxen in VMware Fusion 5 a number of times to make sure the changes I made
were going to do the right thing on a fresh install. I'd recommend going down
THAT road if you're hesitant of immediately throwing this on your pretty
snowflake of a laptop.)

Installing Boxen was pretty easy - the only prerequisite was downloading
the [XCode Command-Line Tools](http://developer.apple.com/library/ios/#documentation/DeveloperTools/Conceptual/WhatsNewXcode/Articles/xcode_4_3.html)
(which included git), pulling down the Boxen repo, and running `script/boxen`.
It was stupid simple. What you GOT, by default, was:

* Homebrew
* Git
* Hub
* DNSMasq w/ .dev resolver for localhost
* NVM
* RBenv
* Full Disk Encryption requirement
* NodeJS 0.4
* NodeJS 0.6
* NodeJS 0.8
* Ruby 1.8.7
* Ruby 1.9.2
* Ruby 1.9.3
* Ack
* Findutils
* GNU-Tar

Remember, this is all tunable and you don't need to pull down ALL of these
packages, but, since it was new, I decided to install everything and sort
it out later.  Yes, the initial setup took a good number of minutes, but
think about everything that's being installed. In the end, I had a full Ruby
development environment with rbenv, multiple versions of Ruby, and a laptop
that could be customized without much work at all.

## Which end do I blow in?

The readme on [the project page for Boxen][boxen] describes how to clone the
project into `/opt/boxen/repo`, so that's the directory we'll be working with.
To see what will be enforced on your machine, check out `manifests/site.pp` to
see something that looks like this:

{% codeblock manifests/site.pp lang:puppet %}
require boxen::environment
require homebrew::repo

Exec {
  group       => 'staff',
  logoutput   => on_failure,
  user        => $luser,

  path => [
    "${boxen::config::home}/rbenv/shims",
    "${boxen::config::home}/homebrew/bin",
    '/usr/bin',
    '/bin',
    '/usr/sbin',
    '/sbin'
  ],  

  environment => [
    "HOMEBREW_CACHE=${homebrew::cachedir}",
    "HOME=/Users/${::luser}"
  ]
}

File {
  group => 'staff',
  owner => $luser
}

Package {
  provider => homebrew,
  require  => Class['homebrew']
}

Repository {
  provider => git,
  extra    => [
    '--recurse-submodules'
  ],
  require  => Class['git']
}

Service {
  provider => ghlaunchd
}
{% endcodeblock %}

This is largely scaffolding setting up the Boxen environment and resource
defaults. If you're familiar with Puppet, this should be recognizable to you,
but for everyone else, let's dissect one of the resource defaults:

{% codeblock lang:puppet %}
File {
  group => 'staff',
  owner => $luser
}
{% endcodeblock %}

This block basically means that any file you declare with Puppet should default
to having its owner set as your username and its group set to 'staff' (which is
standard in OS X). You can override this explicitly with a file declaration by
providing the owner or group attribute, but if you omit it then it's going to
default to these values.

The rest of the defaults are customized for Boxen's preferences (i.e. homebrew
will be used to install all packages unless you specify otherwise, exec
resources will log all output on failure, service resources will use githubs's
customized service provider, and etc...).  Now let's look below:

{% codeblock manifests/site.pp lang:puppet %}
node default {
  # core modules, needed for most things
  include dnsmasq
  include git 
  include hub 
  include nginx
  include nvm 
  include ruby

  # fail if FDE is not enabled
  if $::root_encrypted == false {
    fail('Please enable full disk encryption and try again')
  }

  # node versions
  include nodejs::0-4
  include nodejs::0-6
  include nodejs::0-8

  # default ruby versions
  include ruby::1-8-7
  include ruby::1-9-2
  include ruby::1-9-3

  # common, useful packages
  package {
    [   
      'ack',
      'findutils',
      'gnu-tar'
    ]:  
  }

  file { "${boxen::config::srcdir}/our-boxen":
    ensure => link,
    target => $boxen::config::repodir
  }
}
{% endcodeblock %}

These are the things that Boxen has chosen to enforce 'out of the box'. Knowing
that Boxen was designed so that developers could customize their 'boxes'
THEMSELVES, it makes sense that there's not much that's being enforced on
everyone. In fact, the most significant thing being 'thrust' upon you is the
fact that the machine must have full disk encryption enabled (which is a
good idea anyways).

If you want to pare down what Boxen gives you by default, you can choose to
comment out lines providing, for example, nvm and nodejs versions (if you don't
use node.js in your environment). I'm a Ruby developer, so all the Ruby builds
(and rbenv) are very helpful to me, but you could also remove those if you were
so inclined. The point is that this file contains the 'knobs' to dial your base
Boxen setup up or down.


## Customizing (or, my dotfiles are better than yours)

The whole point of Boxen is to customize your laptop and keep its customization
automated. To do this, we're going to need to make some Puppet class files.

** CAUTION: PUPPET AHEAD **

If you've not had experience with Puppet before, [I can't recommend the learning
Puppet series enough][learnpuppet]. In the vein of "Puppet now, learn later",
I'm going to give you Puppet code that works for ME and only explain the trickier
bits.

Boxen has some 'magic' code that's going to automatically look for a class
called `people::<github username>`, and so I'm going to create a file in
`modules/people/manifests` called `glarizza.pp`.  This file will contain Puppet
code specific to MY laptop(s).  Here's a snippit of that file:

{% codeblock modules/people/manifests/glarizza.pp lang:puppet %}
class people::glarizza {

  notify { 'class people::glarizza declared': }

  # Changes the default shell to the zsh version we get from Homebrew
  # Uses the osx_chsh type out of boxen/puppet-osx
  osx_chsh { $::luser:
    shell   => '/opt/boxen/homebrew/bin/zsh',
    require => Package['zsh'],
  }

  file_line { 'add zsh to /etc/shells':
    path    => '/etc/shells',
    line    => "${boxen::config::homebrewdir}/bin/zsh",
    require => Package['zsh'],
  }

  ##################################
  ## Facter, Puppet, and Envpuppet##
  ##################################

  repository { "${::boxen_srcdir}/puppet":
    source => 'puppetlabs/puppet',
  }

  repository { "${::boxen_srcdir}/facter":
    source => 'puppetlabs/facter',
  }

  file { '/bin/envpuppet':
    ensure  => link,
    mode    => '0755',
    target  => "${::boxen_srcdir}/puppet/ext/envpuppet",
    require => Repository["${::boxen_srcdir}/puppet"],
  }
}
{% endcodeblock %}

The notify resource is only to prove that when you run Boxen that this class
is being declared - it only displays a message to the console when you run
the `boxen` binary.

The `osx_chsh` resource is a custom defined type that Github has created to
ensure a line shows up in `/etc/shells` as an acceptable shell. Because
Boxen installs zsh from homebrew into `/opt/boxen/homebrew`, we need to ensure
that `/etc/shells` is correct. Note the syntax of `$boxen::config::homebrewdir`
which refers to a variable called `$homebrewdir` in the `boxen::config` class.

Next, I've setup a couple of resources to make sure the `puppet` and `facter`
repositories are installed on my system. Github has also developed a lightweight
`repository` resource that will simply ensure that a repo is cloned at a location
on disk. `$::boxen_srcdir` is one of the [custom Facter facts][customfacts] that
Boxen provides in `shared/boxen/lib/facter/boxen.rb` in the Boxen repository.

The file resource sets up a symlink from `/bin/envpuppet` to
`/Users/glarizza/src/puppet/ext/envpuppet` on my system. The attributes should
be pretty self-explanatory, but the newest attribute of `require` says that the
`repository` resource must come BEFORE this file resource is declared. This is
a demonstration of [Puppet's ordering metaparameters][order] that are described
in the [Learning Puppet][learnpuppet] series.

Since we briefly touched on `$::boxen_srcdir`, what are some other custom facts
that come out of `shared/boxen/lib/facter/boxen.rb`?

{% codeblock shared/boxen/lib/facter/boxen.rb lang:ruby %}
require "json"
require "boxen/config"

config   = Boxen::Config.load
facts    = {}
factsdir = File.join config.homedir, "config", "facts"

facts["github_login"]   = config.login
facts["github_email"]   = config.email
facts["github_name"]    = config.name

facts["boxen_home"]     = config.homedir
facts["boxen_srcdir"]   = config.srcdir

if config.respond_to? :reponame
  facts["boxen_reponame"] = config.reponame
end

facts["luser"]          = config.user

Dir["#{config.homedir}/config/facts/*.json"].each do |file|
  facts.merge! JSON.parse File.read file
end

facts.each { |k, v| Facter.add(k) { setcode { v } } } 
{% endcodeblock %}

This file will also give you `$::luser`, which will evaluate out to your system
username, and `$::github_name`, which is equivalent to your Github username
(note that this is what Boxen uses to find your class file in
`modules/people/manifests`). If you're looking for all the other values set by
these custom facts, check out `config/boxen/defaults.json` after you run Boxen.


## Using modules out of the Boxen namespace

Not only is Boxen its own project, but it's a [separate organization on Github
that hosts a number of Puppet modules][boxengithub]. Some of these modules are
pretty simple (a single resource to install a package), but the point is that
they've been provided FOR you - so use, fork, and improve them (but most of
all, submit pull requests). The way you use them with Boxen may not be readily
clear, so let's walk through that with a simple module for installing Google
Chrome.

1. Add the module to your Puppetfile
2. Classify the module in your Puppet setup
3. Run Boxen


#### Add the module to your Puppetfile

Boxen uses a tool called [librarian-puppet][lp] to source and install Puppet
modules from Github. Librarian-puppet uses the `Puppetfile` file in the root of
the Boxen repo to install modules.  Let's look at a couple of lines in that
file:

{% codeblock Puppetfile lang:ruby %}
mod "boxen",    "0.1.8",  :github_tarball => "boxen/puppet-boxen"
mod "dnsmasq",  "0.0.1",  :github_tarball => "boxen/puppet-dnsmasq"
mod "git",      "0.0.3",  :github_tarball => "boxen/puppet-git"
mod "hub",      "0.0.1",  :github_tarball => "boxen/puppet-hub"
mod "homebrew", "0.0.17", :github_tarball => "boxen/puppet-homebrew"
mod "inifile",  "0.0.1",  :github_tarball => "boxen/puppet-inifile"
mod "nginx",    "0.0.2",  :github_tarball => "boxen/puppet-nginx"
mod "nodejs",   "0.0.2",  :github_tarball => "boxen/puppet-nodejs"
mod "nvm",      "0.0.5",  :github_tarball => "boxen/puppet-nvm"
mod "ruby",     "0.4.0",  :github_tarball => "boxen/puppet-ruby"
mod "stdlib",   "3.0.0",  :github_tarball => "puppetlabs/puppetlabs-stdlib"
mod "sudo",     "0.0.1",  :github_tarball => "boxen/puppet-sudo"
{% endcodeblock %}

This evaluates out to the following syntax:

```
mod, <module name>, <version or tag>, <source>
```

The **HARDEST** thing about this file is finding the version number of modules
on Github (HINT: it's a tag). Once you're given that information, it's easy to
pull up a module on Github, look at its tags, and then fill out the file. Let's
do that with a line for the Chrome module:

{% codeblock lang:ruby %}
mod "chrome",     "0.0.2",   :github_tarball => "boxen/puppet-chrome"
{% endcodeblock %}

#### Classify the module in your Puppet setup

In the previous section, we created `modules/people/manifests/<github
username>.pp`.  We COULD continue to fill this file with a ton of resources,
but I tend to like to separate out resources into separate subclasses. [Puppet
has module naming conventions to ensure that it can FIND your
subclasses][modulenaming], so I recommend browsing that guide before randomly
naming files (**HINT:** Filenames ARE important and DO matter here). I want to
create a `people::glarizza::applications` subclass, so I need to do the
following:

```
## YES, make sure to replace YOUR USERNAME for 'glarizza'
$ cd /opt/boxen/repo
$ mkdir -p modules/people/manifests/glarizza
$ vim modules/people/manifests/glarizza/applications.pp
```

It's totally fine that there's a `glarizza` directory aside the `glarizza.pp`
file - this is intentional and desired. Puppet's not going to automatically
declare anything in the `people::glarizza::applications` class until we TELL it
to, so let's open `modules/people/manifests/glarizza.pp` and add the following
line at the top:

{% codeblock modules/people/manifests/glarizza.pp lang:puppet %}
include people::glarizza::applications
{% endcodeblock %}

That tells Puppet to find the `people::glarizza::applications` class and make
sure it 'does' everything in that file. Now, let's create the
`people::glarizza::applications` class:

{% codeblock modules/people/manifests/glarizza/applications.rb lang:puppet %}
class people::glarizza::applications {
  include chrome
}
{% endcodeblock %}

Yep, all it takes is one line to include the module we will get from Boxen.
Because of the way Boxen works, it will consult the Puppetfile FIRST, pull down
any modules that are in the Puppetfile but NOT on the system, drop them into
place so Puppet can find them, and then run Puppet normally.


#### Run Boxen

Once you have Boxen setup, you can just run `boxen` from the command line to
have it enforce your configuration. By default, if there are any errors, it
will log them as Github Issues on your fork of the main Boxen repository (this
can be disabled with `boxen --no-issue`). As you're just getting started,
don't worry about the errors. The good news is that once you fix things and
perform a successful Boxen run, it will automatically close all open issues.
If everything went well, you should now have Google Chrome in your
`/Applications` directory!


## ¡Más Puppet!

You'll find as you start customizing all the things that you're usually
managing one of the following resources:

1. Packages
2. Files
3. Repositories
4. Plist files

We've covered managing a repository and a file, but let's look at a couple of
the other popular resources:

### Packages are annoying

I would be willing to bet that most of the things you end up managing will be
packages. Using Puppet with Boxen, you have the ability to install four
different kinds of packages:

1. Applications inside a DMG
2. Installer packages inside a DMG
3. Homebrew Packages
4. Applications inside a .zip file

Here's an example of every type of package installer:

{% codeblock lang:puppet %}
  # Application in a DMG
  package { 'Gephi':
    ensure   => installed,
    source   => 'https://launchpadlibrarian.net/98903476/gephi-0.8.1-beta.dmg',
    provider => appdmg,
  }

  # Installer in a DMG
  package { 'Virtualbox':
    ensure => installed,
    source => 'http://download.virtualbox.org/virtualbox/4.1.22/VirtualBox-4.1.23-80870-OSX.dmg',
    provider => pkgdmg,
  }

  # Homebrew Package
  package { 'tmux':
    ensure => installed,
  }

  # Application in a .zip
  package { 'Github for Mac':
    ensure   => installed,
    source   => 'https://github-central.s3.amazonaws.com/mac%2FGitHub%20for%20Mac%2069.zip',
    provider => compressed_app
  }
{% endcodeblock %}

Notice that the only thing that changes among these resources is the `provider`
attribute. Remember from before that Boxen sets the default package provider to
be 'homebrew', so for 'tmux' I omitted the provider attribute to utilize the
default. Also, the `ensure` attribute is defaulted to 'installed', so
technically I could remove it...but I tend to prefer to use it for people who
will be reading my code later.

There's no provider for .pkg files. Why? Well, packages on OS X are
either bundles or flat-packages. Bundles LOOK like individual files, but they're
actually folders that contain everything necessary to expand and install the
package. Flat packages are just that - an actual file that ends in .pkg that
can be expanded to install whatever you want. Bundle packages are pretty
common, but they're also hard for curl to download them (being that it's just
a folder full of files) - this is why most installer packages you encounter on
OS X are going to be enclosed in a .dmg Disk Image.

So which provider will you use? Well, if your file ends in .dmg then you're
going to be using either the `pkgdmg` or `appdmg` provider. How do you know
which to use? Expand the .dmg file and look inside it. If it contains an
application ending in .app that simply needs dragged into the `/Applications`
folder on disk, then chose the `appdmg` provider (that's essentially all it
does - expand the .dmg file and ditto the .app file into `/Applications`).
If the disk image contains a .pkg package installer, then you'll chose the
`pkgdmg` provider (which expands the .dmg file and uses `installer` to install
the contents of the .pkg file silently in the background). If your file is a
.zip file containing an Application (.app file), then you can use Github's
custom `compressed_app` provider that will unzip the file and ditto the app
into `/Applications`. Finally, if you want to install a package from Homebrew,
then the `homebrew` provider is pretty self-explanatory here.

(NOTE: There is ONE more package provider I haven't covered here - the `macports`
provider. It requires [Macports](http://www.macports.org) to be installed on your
system, and will use it to install a package. Macports vs. Homebrew arguments
notwithstanding, if you're into Macports then there's a provider for you.)


### Plists: because why NOT XML :\

Apple falls somewhere between "the registry" and "config files" on the timeline
of tweaking system settings. Most settings are locked up in plist files that
can be managed by hand or with `plistbuddy` or `defaults`. A couple of people
have saved their customizations in with their dotfiles ([Zach Holman has an example here][holmansettings]),
but Puppet is a great way for managing individual keys in your plist files.
[I've written a module that will manage any number of keys in a plist
file][plist].  You can modify your `Puppetfile` to
make sure Boxen picks up my module by adding the following line:

{% codeblock lang:ruby %}
mod "property_list_key",  "0.1.0",   :github_tarball => "glarizza/puppet-property_list_key"
{% endcodeblock %}

Next, you'll need to add resources to your classes:

{% codeblock lang:puppet %}
  # Disable Gatekeeper so you can install any package you want
  property_list_key { 'Disable Gatekeeper':
    ensure => present,
    path   => '/var/db/SystemPolicy-prefs.plist',
    key    => 'enabled',
    value  => 'no',
  }

  $my_homedir = "/Users/${::luser}"

  # NOTE: Dock prefs only take effect when you restart the dock
  property_list_key { 'Hide the dock':
    ensure     => present,
    path       => "${my_homedir}/Library/Preferences/com.apple.dock.plist",
    key        => 'autohide',
    value      => true,
    value_type => 'boolean',
    notify     => Exec['Restart the Dock'],
  }

  property_list_key { 'Align the Dock Left':
    ensure     => present,
    path       => "${my_homedir}/Library/Preferences/com.apple.dock.plist",
    key        => 'orientation',
    value      => 'left',
    notify     => Exec['Restart the Dock'],
  }

  property_list_key { 'Lower Right Hotcorner - Screen Saver':
    ensure     => present,
    path       => "${my_homedir}/Library/Preferences/com.apple.dock.plist",
    key        => 'wvous-br-corner',
    value      => 10,
    value_type => 'integer',
    notify     => Exec['Restart the Dock'],
  }

  property_list_key { 'Lower Right Hotcorner - Screen Saver - modifier':
    ensure     => present,
    path       => "${my_homedir}/Library/Preferences/com.apple.dock.plist",
    key        => 'wvous-br-modifier',
    value      => 0,
    value_type => 'integer',
    notify     => Exec['Restart the Dock'],
  }

  exec { 'Restart the Dock':
    command     => '/usr/bin/killall -HUP Dock',
    refreshonly => true,
  }

  file { 'Dock Plist':
    ensure  => file,
    require => [
                 Property_list_key['Lower Right Hotcorner - Screen Saver - modifier'],
                 Property_list_key['Hide the dock'],
                 Property_list_key['Align the Dock Left'],
                 Property_list_key['Lower Right Hotcorner - Screen Saver'],
                 Property_list_key['Lower Right Hotcorner - Screen Saver - modifier'],
               ],
    path    => "${my_homedir}/Library/Preferences/com.apple.dock.plist",
    mode    => '0600',
    notify     => Exec['Restart the Dock'],
  }
{% endcodeblock %}

The important attributes are:

1. `path`: The path to the plist file on disk
2. `key`: The individual KEY in the plist file you want to manage
3. `value`: The value that the key should have in the plist file
4. `value_type`: The datatype the value should be (defaults to `string`, but
   could also be `array`, `hash`, `boolean`, or `integer`)

You MUST pass a path, key, and value or Puppet will throw an error.

The first resource above sets Gatekeeper in 10.8 and allows you to install
packages from the web that HAVEN'T been signed (in 10.8, Apple won't allow you
to install unsigned packages or anything outside of the App Store without
enabling this setting).

All of the other resources relate to making changes to the Dock. Because of the
way the Dock is managed, you must `HUP` its process when making changes to your
dock plist before they take effect. Also, the dock plist has to be owned by
you or else the changes won't take effect. Every dock plist resource has
a `notify` metaparameter which means "any time this resource changes, run this
exec resource". That exec resource is a simple command that `HUP`s the dock
process.  It will ONLY be run if a resource notifies it - so if no changes are
made in a Puppet run then the command won't fire. Finally, the file resource to
manage the dock plist ensures that permissions are set (and notifies the exec
in case it needs to CHANGE permissions).

Again, this is purely dealing with Puppet - but plists are a major part of OS
X and you'll be dealing with them regularly!


### But seriously, dotfiles

I know I've joked about it a couple of times, but getting your dotfiles into
their correct location is a quick win. The secret is to lock them all up in
a repository, and then symlink them where you need them. Let's look at that:

{% codeblock lang:puppet %}
  # My dotfile repository
  repository { "${my_sourcedir}/dotfiles":
    source => 'glarizza/dotfiles',
  }

  file { "${my_homedir}/.tmux.conf":
    ensure  => link,
    mode    => '0644',
    target  => "${my_sourcedir}/dotfiles/tmux.conf",
    require => Repository["${my_sourcedir}/dotfiles"],
  }

  file { "/Users/${my_username}/.zshrc":
    ensure  => link,
    mode    => '0644',
    target  => "${my_sourcedir}/dotfiles/zshrc",
    require => Repository["${my_sourcedir}/dotfiles"],
  }

  file { "/Users/${my_username}/.vimrc":
    ensure => link,
    mode   => '0644',
    target => "${my_sourcedir}/dotfiles/vimrc",
    require => Repository["${my_sourcedir}/dotfiles"],
  }

  # Yes, oh-my-zsh. Judge me.
  file { "/Users/${my_username}/.oh-my-zsh":
    ensure  => link,
    target  => "${my_sourcedir}/oh-my-zsh",
    require => Repository["${my_sourcedir}/oh-my-zsh"],
  }
{% endcodeblock %}

It's worth mentioning that Puppet does not do things procedurally. Just because
the dotfiles repository is listed before every symlink DOES NOT mean that Puppet
will evaluate and declare it first. You'll need to specify order here, and that's
what the `require` metaparameter does.

Based on what I've already shown you, this code block should be very simple to
follow. Because I'm using symlinks, the dotfiles should always be current.
Because the dotfiles are under revision control, updating them all is as simple
as making commits and updating your repository. If you've ever had to migrate
these files to a new VM/machine, then you know how full of win this block of
code is.


## Don't sweat petty (or pet sweaty)

When I show sysadmins/developers automation like this, they usually want to
apply it to the HARDEST part of their day-job IMMEDIATELY. That's a somewhat
rational reaction, but it's not going to give you the results you want. The
cool thing ABOUT Boxen and Puppet is that it's going to remove those little
annoyances in your day that slowly sap your time. START by tackling those
small annoyances to remove them and build your confidence (like the dotfiles
example above). Yeah, you'll only save a couple of minutes a day, but it
grows exponentially. Also, when you solve a problem during the course of your
day, MANAGE it with Boxen by putting it in your Puppet class (then, test it
out on a VM or another machine to make sure it does what you expect).


Don't worry that you're not saving the world with a massive Puppet class -
sometimes the secret to happiness is opening iTerm on a new machine and
seeing your finely-crafted prompt shining in its cornflower-blue glory.


## Now show me some cool stuff

So that's a quick tour of the basics of Boxen and the kinds of things you can
do from the start.  I'm really excited for everyone to get their hands on Boxen
and do more cool stuff with Puppet. I've done a bunch of work with Puppet for
OS X, and that's enough to know that there's still PLENTY that can be
improved in the codebase.  A giant THANK YOU to [John Barnette][jb], [Will Farrington][wf],
and the entire Github Boxen team for all their work on this
tool (and letting me tinker with it before it hit the general public)! Feel
free to comment below, email me (gary at puppetlabs), or [yell at me on Twitter][gl]
for more information!


[boxen]: http://boxen.github.com
[learnpuppet]: http://docs.puppetlabs.com/learning/
[customfacts]: http://docs.puppetlabs.com/guides/custom_facts.html
[order]: http://docs.puppetlabs.com/learning/ordering.html
[boxengithub]: http://github.com/boxen
[lp]: https://github.com/rodjek/librarian-puppet
[modulenaming]: http://docs.puppetlabs.com/learning/modules1.html#manifests-namespacing-and-autoloading
[holmansettings]: https://github.com/holman/dotfiles/blob/master/osx/set-defaults.sh
[plist]: http://github.com/glarizza/puppet-property_list_key
[jb]: http://twitter.com/jbarnette
[wf]: http://twitter.com/wfarr
[gl]: http://twitter.com/glarizza
