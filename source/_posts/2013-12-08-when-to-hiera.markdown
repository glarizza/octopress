---
layout: post
title: "When to Hiera (aka: How do I Module?)"
date: 2013-12-08 15:31:54 -0800
comments: true
categories:  ['puppet', 'hiera', 'linux']
---

I'm convinced that writing Puppet modules is the ultimate exercise in bikeshedding:
if it works, someone's probably going to tell you that you could have done it better,
if you're using the methods suggested today, they're probably going to be out-of-date
in about 6 months, and good luck writing something that someone else can use cleanly
without needing to change it.

I can help you with the last two.

## Data and Code Separation == bliss?

[I wrote a blog post about 2 years ago](http://bit.ly/puppetdata) detailing why separating
your data from your Puppet code was a good idea. The idea is still valid, which means
it's probably one of the better ideas I've ever stolen (Does anyone want any HD-DVDs?).
Hunter Haugen and I [put together a quick blog post on using Hiera](http://bit.ly/hierablog)
to solve the data/code problem because there wasn't a great bit of documentation on Hiera
at that point in time. Since then, Hiera's been widely accepted as "a good idea" and is
in use in production Puppet environments around the world. In most every environment,
usage of Hiera by more than just one person eventually gives birth to the question
that inspired this post:

#### "What the hell does and does NOT belong in Hiera?"

## Puppet data models

### The params class pattern

Many Puppet modules out there since Puppet 2.6 have begun using this pattern:

{% codeblock lang:puppet puppetlabs-mysql/manifests/server.pp %}
class mysql::server (
  $config_file             = $mysql::params::config_file,
  $manage_config_file      = $mysql::params::manage_config_file,
  $old_root_password       = $mysql::params::old_root_password,
  $override_options        = {},
  $package_ensure          = $mysql::params::server_package_ensure,
  $package_name            = $mysql::params::server_package_name,
  $purge_conf_dir          = $mysql::params::purge_conf_dir,
  $remove_default_accounts = false,
  $restart                 = $mysql::params::restart,
  $root_group              = $mysql::params::root_group,
  $root_password           = $mysql::params::root_password,
  $service_enabled         = $mysql::params::server_service_enabled,
  $service_manage          = $mysql::params::server_service_manage,
  $service_name            = $mysql::params::server_service_name,
  $service_provider        = $mysql::params::server_service_provider,
  # Deprecated parameters
  $enabled                 = undef,
  $manage_service          = undef
) inherits mysql::params {

  ## Puppet goodness goes here
}

{% endcodeblock %}

If you're not familiar, this is a Puppet class definition for `mysql::server` that has several parameters
defined and defaulted to values that come out of the `mysql::params` class.  The
`mysql::params` class looks a bit like this:

{% codeblock lang:puppet puppetlabs-mysql/manifests/params.pp %}
class mysql::params {
  case $::osfamily {
    'RedHat': {
      if $::operatingsystem == 'Fedora' and (is_integer($::operatingsystemrelease) and $::operatingsystemrelease >= 19 or $::operatingsystemrelease == "Rawhide") {
        $client_package_name = 'mariadb'
        $server_package_name = 'mariadb-server'
      } else {
        $client_package_name = 'mysql'
        $server_package_name = 'mysql-server'
      }
      $basedir             = '/usr'
      $config_file         = '/etc/my.cnf'
      $datadir             = '/var/lib/mysql'
      $log_error           = '/var/log/mysqld.log'
      $pidfile             = '/var/run/mysqld/mysqld.pid'
      $root_group          = 'root'
    }

    'Debian': {
      ## More parameters defined here
    }
  }
}

{% endcodeblock %}

This pattern puts all conditional logic for all the variables/parameters used
in the module inside one class - the `mysql::params` class.  It's called the
'params class pattern' because we suck at naming things.

#### Pros:
* All conditional logic is in a single class
* You always know which class to seek out if you need to change any of the logic used to determine a variable's value
* You can use the include function because parameters for each class will be defaulted to the values that came out of the params class
* If you need to override the value of a particular parameter, you can still use the parameterized class declaration syntax to do so
* Anyone using Puppet version 2.6 or higher can use it (i.e. anyone who's been using Puppet since about 2010).

##### Cons:

* Conditional logic is repeated in every module
* You will need to use inheritance to inherit parameter values in each subclass
* It's another place to look if you ALSO use Hiera inside the module
* Data is inside the manifest, so business logic is also inside params.pp

### Hiera defaults pattern

When Hiera hit the scene, one of the first things people tried to do was
to incorporate it into existing modules. The logic at that time was that
you could keep all parameter defaults inside Hiera, rid yourself of the
params class, and then just make Hiera calls out for your data. This
pattern looks like this:

{% codeblock lang:puppet puppetlabs-mysql/manifests/server.pp %}
class mysql::server (
  $config_file             = hiera('mysql::params::config_file', 'default value'),
  $manage_config_file      = hiera('mysql::params::manage_config_file', 'default value'),
  $old_root_password       = hiera('mysql::params::old_root_password', 'default value'),
  ## Repeat the above pattern
) {

  ## Puppet goodness goes here
}

{% endcodeblock %}

#### Pros:

* All data is locked up in Hiera (and its multiple backends)
* Default values can be provided if a Hiera lookup fails

#### Cons:

* You need to have Hiera installed, enabled, and configured to use this pattern
* All data, including non-business logic, is in Hiera
* If you use the default value, data could either come from Hiera OR the default (multiple places to look when debugging)


### Hybrid data model

This pattern is for those people who want the portability of the params.pp class
combined with the power of Hiera. Because it's a hybrid, there are multiple ways
that people have set it up.  Here's a general example:

{% codeblock lang:puppet puppetlabs-mysql/manifests/server.pp %}
class mysql::server (
  $config_file             = hiera('mysql::params::config_file', $mysql::params::config_file),
  $manage_config_file      = hiera('mysql::params::manage_config_file', $mysql::params::manage_config_file),
  $old_root_password       = hiera('mysql::params::old_root_password', $mysql::params::old_root_password),
  ## Repeat the above pattern
) inherits mysql::params {

{% endcodeblock %}

#### Pros:
* Data is sought from Hiera first and then defaulted back to the params class parameter
* Keep non-business logic (i.e. OS specific data) in the params class and business logic in Hiera
* Added benefits of both models

#### Cons:
* Where did the variable get set - Hiera or the params class? Debugging can be hard
* Requires Hiera to be setup to use the module
* If you fudge a variable name in Hiera, you get the params class default - see Con #1

### Hiera data bindings in Puppet 3.x.x

In Puppet 3.0.0, there was a concept introduced called Data Bindings. This created
a federated data model automatically incorporating a Hiera lookup. Previously, the
order that Puppet would use to determine the value of a parameter was to first use
a value passed with the parameterized class declaration syntax (i.e. the below:).

{% codeblock lang:puppet parameterized class declaration %}
class { 'apache':
  package_name => 'httpd',
}
{% endcodeblock %}

If a parameter was not passed with the parameterized class syntax (like the 'package_name'
parameter above'), Puppet would then look for a default value inside the class definition
(i.e. the below:).

{% codeblock lang:puppet parameter default in a class definition %}
class ntp (
  $ntpserver = 'default.ntpserver.org'
) {
  # Use $ntpserver in a file declaration...
}
{% endcodeblock %}

If the value of 'ntpserver' wasn't passed with a parameterized class declaration,
then the value would be set to 'default.ntpserver.org', since that's the default
set in the above class definition.

Failing both of these conditions, Puppet would throw a parse error and say
that it couldn't determine a value for a class parameter.

As of Puppet 3.0.0, Puppet will now do a Hiera lookup for the fully namespaced
value of a class parameter


### Roles and Profiles

[The roles and profiles pattern](http://sysadvent.blogspot.com/2012/12/day-13-configuration-management-as-legos.html) has been written about
a number of times and is ALSO considered to be 'a best practice' when setting
up your Puppet environment. What roles and profiles gets you is a 'wrapper
class' that allows you to declare classes within this wrapper class:

{% codeblock lang:puppet profiles/manifests/wordpress.pp %}

class profiles::wordpress {
  # Data Lookups
  $site_name               = hiera('profiles::wordpress::site_name')
  $wordpress_user_password = hiera('profiles::wordpress::wordpress_user_password')
  $mysql_root_password     = hiera('profiles::wordpress::mysql_root_password')
  $wordpress_db_host       = hiera('profiles::wordpress::wordpress_db_host')
  $wordpress_db_name       = hiera('profiles::wordpress::wordpress_db_name')
  $wordpress_db_password   = hiera('profiles::wordpress::wordpress_db_password')

  ## Create user
  group { 'wordpress':
    ensure => present,
  }
  user { 'wordpress':
    ensure   => present,
    gid      => 'wordpress',
    password => $wordpress_user_password,
    home     => '/var/www/wordpress',
  }

  ## Configure mysql
  class { 'mysql::server':
    root_password => $wordpress_root_password,
  }

  class { 'mysql::bindings':
    php_enable => true,
  }

  ## Configure apache
  include apache
  include apache::mod::php
}

## Continue with declarations...

{% endcodeblock %}

Notice that any variables that might have business specific logic are set with
Hiera lookups. These Hiera lookups do NOT have default values, which means the
`hiera()` function will throw a parse error if a value is not returned. This
is IDEAL because we WANT TO KNOW if a Hiera lookup fails - this means we failed
to put the data in Hiera and should be corrected BEFORE a state that might
contain invalid data is enforced with Puppet.

You then have a 'Role' wrapper class that simply includes many of the 'Profile'
wrapper classes:

{% codeblock lang:puppet roles/manifests/frontend.pp %}
class roles::frontend {
  include profiles::mysql
  include profiles::apache
  include profiles::java
  include profiles::jboss
  # include more profiles...
}
{% endcodeblock %}

The idea being that Profiles abstract all the technical bits that need to
declared to setup a piece of technology, and Roles will abstract all the
business logic for what pieces of technology should be installed on a certain
'class' of machine.  Basically, you can say that "all our frontend infrastructure
should have mysql, apache, java, jboss...".  In this statement, the Role is
'frontend infrastructure' and the Profiles are 'mysql, apache, java, jboss...'.


#### Pros:
* Hiera data lookups are confined to a wrapper class OUTSIDE of the component modules (like mysql, apache, java, etc...)
* Data lookups for parameters containing business logic are done with Hiera
* Non-business specific data is pulled from the module (i.e. the params class)
* Wrapper modules can be 'included' with the `include` function, helping to eliminate multiple class declarations using the parameterized class declaration syntax
* Component modules are backward-compatible to Puppet 2.6 while wrapper modules still get to use a modern data lookup mechanism (Hiera)
* Component modules do NOT contain any business specific logic, which means they're portable

#### Cons:
* Hiera must be setup to use the wrapper modules
* Wrapper modules add another debug path for variable data
* Wrapper modules add another layer of abstraction

## Data in Puppet Modules

R.I. Pienaar (the original author of MCollective, Hiera, and much more)
[published a blog post](http://www.devco.net/archives/2013/12/08/better-puppet-modules-using-hiera-data.php)
recently on implementing a folder for Puppet modules that Hiera can traverse
when it does data lookups. This construct isn't new,
[there was a feature request](https://projects.puppetlabs.com/issues/16856)
for this behavior filed in October of 2012 with a
[subsequent pull request](https://github.com/puppetlabs/puppet/pull/1217)
that implemented this functionality (they're both worth reads for further
information). The pull request didn't get merged, and so R.I. implemented
the functionality [inside a module on the Puppet Forge](http://forge.puppetlabs.com/ripienaar/module_data).
In a nutshell, it's a hiera.yaml configuration file INSIDE THE MODULE that
implements a module-specific hierarchy, and a 'data' folder (also inside
the module) that allows for individual YAML files that Hiera could read.
This hierarchy is consulted AFTER the site-specific `hiera.yaml` file
is read (i.e. `/etc/puppet/hiera.yaml` or `/etc/puppetlabs/puppet/hiera.yaml`),
and the in-module data files are consulted AFTER the site-specific Hiera
data files are read (normally found in either `/etc/puppet/hieradata` or
`/etc/puppetlabs/puppet/hieradata`).

The argument here is that there's a data store for **SITE-SPECIFIC** Hiera
data that should be kept outside of modules, but there's not a **MODULE-SPECIFIC**
data store that Hiera can use. The argument isn't whether data that should be
shared with other people belongs inside a site-specific Hiera datastore
(protip: it doesn't. Data that's not business-specific should be shared
with others and kept inside the module), the argument is that it shouldn't
be locked up inside the DSL where the barrier-to-entry is learning Puppet's
DSL syntax. Whereas `/etc/puppet/hiera.yaml` or `/etc/puppetlabs/puppet/hiera.yaml`
sets up the hierarchy for all your site-specific data, there's no per-module
`hiera.yaml` file for all module-specific data, and there's no place to put
module-specific Hiera data.

#### But module-specific data goes inside the params class and business-specific data goes inside Hiera, right?

Sure, but for some people the Puppet DSL is a barrier. The argument is that
there should be a lower barrier to entry to contribute parameter data
to Puppet that doesn't require you to learn the syntax of if/case/selector
statements in the Puppet DSL. There's also the argument that if you want
to add support for an operatingsystem to your module, you have to modify
the params class file and add another entry to the if/case/selector
statement - wouldn't it be easier to just add another YAML file into
a data folder that doesn't affect existing datafiles?

#### Great, ANOTHER hierarchy to traverse for data - that's going to get confusing

Well, think about it right now - most EVERY params class of EVERY module
(if it supports multiple operatingsystems)
does some sort of conditional logic to determine values for parameters
on a per-OS basis. That's something that you need to traverse. And many
modules use different conditional data to determine what paramters to use. Look
at the mysql params class example above - it not only splits on
`$osfamily`, but it also checks specific operatingsystems. That's a
conditional inside a conditional. You're TRAVERSING conditional data
right now to find a value - the only difference is that this method
doesn't use the DSL, it uses Hiera and YAML.

#### Sure, but this is outside of Puppet and you're losing visibility inside Puppet with your data

You're already doing that if you're using the params class. In this case, visibility is
moved to YAML files instead of separate Puppet classes.


### Setting it up

You will first need to install R.I.'s module from the Puppet Forge. As of
this writing, it's version `0.0.1`, so ensure you have the most recent
version using the `puppet module` tool:

    [root@linux modules]# puppet module install ripienaar/module_data
    Notice: Preparing to install into /etc/puppetlabs/puppet/modules ...
    Notice: Downloading from https://forge.puppetlabs.com ...
    Notice: Installing -- do not interrupt ...
    /etc/puppetlabs/puppet/modules
    └── ripienaar-module_data (v0.0.1)

Next, you'll need to setup a module to use the data-in-modules pattern. Take
a look at the tree of a sample module:

    [root@linux modules]# tree mysql/
    mysql/
    ├── data
    │   ├── hiera.yaml
    │   └── RedHat.yaml
    └── manifests
        └── init.pp

I created a sample `mysql` module based on the examples above. All of
the module's Hiera data (including the module-specific hiera.yaml file)
goes in the `data` folder. This module should be placed in Puppet's
modulepath - and if you don't know where Puppet's modulepath is set,
run the `puppet config` face to determine that:

    [root@linux modules]# puppet config print modulepath
    /etc/puppetlabs/puppet/modules:/opt/puppet/share/puppet/modules

In my case, I'm putting the module in `/etc/puppetlabs/puppet/modules`
(since I'm running Puppet Enterprise). Here's the hiera.yaml file
from the sample mysql module:

{% codeblock lang:yaml mysql/data/hiera.yaml %}
:hierarchy:
  - "%{::osfamily}"
{% endcodeblock %}

I've also included a YAML file for the `$osfamily` of RedHat:

{% codeblock lang:yaml mysql/data/RedHat.yaml %}
---
mysql::config_file: '/path/from/data_in_modules'
mysql::manage_config_file: true
mysql::old_root_password: 'password_from_data_in_modules'
{% endcodeblock %}

Finally, here's what the mysql class definition looks like from
`manifests/init.pp`:

{% codeblock lang:puppet mysql/manifests/init.pp %}
class mysql (
  $config_file        = 'module_default',
  $manage_config_file = 'module_default',
  $old_root_password  = 'module_default'
) {
  notify { "The value of config_file: ${config_file}": }
  notify { "The value of manage_config_file: ${manage_config_file}": }
  notify { "The value of old_root_password: ${old_root_password}": }
}
{% endcodeblock %}

Everything should be setup to notify the value of a couple of parameters.
Now, to test it out...

### Testing data-in-modules

Let's include the mysql class with `puppet apply` to see where it's looking
for data:

    [root@linux modules]# puppet apply -e 'include mysql'
    Notice: The value of config_file: /path/from/data_in_modules
    Notice: /Stage[main]/Mysql/Notify[The value of config_file: /path/from/data_in_modules]/message: defined 'message' as 'The value of config_file: /path/from/data_in_modules'
    Notice: The value of manage_config_file: true
    Notice: /Stage[main]/Mysql/Notify[The value of manage_config_file: true]/message: defined 'message' as 'The value of manage_config_file: true'
    Notice: The value of old_root_password: password_from_data_in_modules
    Notice: /Stage[main]/Mysql/Notify[The value of old_root_password: password_from_data_in_modules]/message: defined 'message' as 'The value of old_root_password: password_from_data_in_modules'
    Notice: Finished catalog run in 0.62 seconds

Since I'm running on an operatingsystem whose family is 'RedHat' (i.e. CentOS),
you can see that the values of all the parameters were pulled from the Hiera
data files inside the module.  Let's temporarily change the `$osfamily` fact
value and see what happens:

    [root@linux modules]# FACTER_osfamily=Debian puppet apply -e 'include mysql'
    Notice: The value of config_file: module_default
    Notice: /Stage[main]/Mysql/Notify[The value of config_file: module_default]/message: defined 'message' as 'The value of config_file: module_default'
    Notice: The value of old_root_password: module_default
    Notice: /Stage[main]/Mysql/Notify[The value of old_root_password: module_default]/message: defined 'message' as 'The value of old_root_password: module_default'
    Notice: The value of manage_config_file: module_default
    Notice: /Stage[main]/Mysql/Notify[The value of manage_config_file: module_default]/message: defined 'message' as 'The value of manage_config_file: module_default'
    Notice: Finished catalog run in 0.51 seconds

This time, when I specified a value of `Debian` for `$osfamily`, the parameter
values were pulled from the declaration in the mysql class definition (i.e.
from inside `mysql/manifests/init.pp`).

### Testing outside of Puppet

One of the big pros of Hiera is that it comes with the `hiera` binary that can
be run from the command line to test values. This works just fine for site-specific
module data that's defined in the central `hiera.yaml` file that's usually defined
in `/etc/puppet` or `/etc/puppetlabs/puppet`, but the data-in-modules pattern relies
on a Puppet indirector to point to the current module's `data` folder, and thus
(as of right now) there's not a simple way to run the `hiera` binary to pull
data out of modules WITHOUT running Puppet. This is not a dealbreaker, and
doesn't stop anybody from hacking up something that WILL look inside modules
for data, but as of right now it doesn't yet exist. It also makes debugging for
values that come out of modules a bit more difficult.


### The scorecard for data-in-modules
#### Pros:

* Parameters are defined in YAML and not Puppet DSL (i.e. you only need to know YAML and not the Puppet DSL)
* Adding parameters is as simple as adding another YAML file to the module
* Module authors provide module data that can be read by Puppet 3.x.x Hiera data bindings

#### Cons:

* Must be using Puppet 3.0.0 or higher
* Additional hierarchy and additional Hiera data file to consult when debugging
* Not (currently) an easy/straightforward way to use the `hiera` binary to test values
* Currently depends on a Puppet Forge module being installed on your system


## What are you trying to say?

I am ALL ABOUT code portability, re-usability, and not building 500 apache modules.
Ever since people have been building modules, they've been putting too much data
inside modules (to the point where they can't share them with anyone else). I
can't tell you how many times I've heard "We have a module for that, but I can't
share it because it has all our company-specific data in it."

Conversely, I've also seen organizations put EVERYTHING in their site-specific
Hiera datastore because "that's the place for Puppet data." They usually end up
with 15+ levels in their Hiera hierarchies because they're doing things like
this:

{% codeblock lang:yaml hiera.yaml %}
---
:backends:
  - yaml

:hierarchy:
  - "%{clientcert}"
  - "%{environment}"
  - "%{osfamily}"
  - "%{osfamily}/%{operatingsystem}"
  - "%{osfamily}/%{operatingsystem}/%{os_version_major}"
  - "%{osfamily}/%{operatingsystem}/%{os_version_minor}"
  # Repeat until you have 15 levels of WTF
{% endcodeblock %}

This leads us back again to "What does and DOESN'T go in Hiera?"
I usually say the following:

#### Data in site-specific Hiera datastore

* Business-specific data (i.e. internal NTP server, VIP address, per-environment java application versions, etc...)
* Sensitive data
* Data that you don't want to share with anyone else

#### Data that does NOT go in the site-specific Hiera datastore

* OS-specific data
* Data that EVERYONE ELSE who uses this module will need to know (paths to config files, package names, etc...)

Basically, if I ask you if I can publish your module to [the Puppet Forge](http://forge.puppetlabs.com),
and you object because it has business-specific or sensitive data in it, then
you probably need to pull that data out of the module and put it in Hiera.

The recommendations that I give when I go on-site with Puppet users is the following:

* Use Roles/Profiles to create wrapper-classes for class declaration
* Do ALL Hiera lookups for site-specific data inside your 'Profile' wrapper classes
* All module-specific data (like paths to config files, names of packages to install, etc...) should be kept in the module in the params class
* All 'Role' wrapper classes should just **include** 'Profile' wrapper classes - nothing else

## But what about Data in Modules?

I went through all the trouble of writing up the Data in Modules pattern,
but I didn't recommend or even MENTION it in the previous section. The reason
is NOT because I don't believe in it (I actually think the future will be data
outside of the DSL inside a Puppet module), the reason is because it's not **YET**
in Puppet's core and because it's not **YET** been widely tested. If you're an
existing Puppet user that's been looking for a way to split data outside of
the DSL, here is your opportunity. Use the pattern and PLEASE report back on what
you like and don't like about it. The functionality is in a module, so it's
easy to tweak. If you're new to Puppet and are comfortable with the DSL, then
the params class exists and is available to you.

To voice your opinion or to follow the progress of data in modules,
[follow or comment on this Puppet ticket.](https://projects.puppetlabs.com/issues/16856)

## Update

[R.I. posted another article on the problem with params.pp](http://www.devco.net/archives/2013/12/09/the-problem-with-params-pp.php)
that is worth reading. He gives compelling reasons on why he built Hiera, why
params.pp WORKS, but also why he believes it's not the future of Puppet. R.I.
goes even further to explain that it's not necessarily the Puppet DSL that is
the barrier to entry, it's that this sort of data belongs in a file for config
data and not INSIDE THE CODE itself (i.e. inside the Puppet DSL). Providing data
inside modules gives module authors a way to provide this configuration data
in files that AREN'T the Puppet DSL (i.e. not inside the code).

