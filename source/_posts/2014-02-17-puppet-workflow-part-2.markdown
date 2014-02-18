---
layout: post
title: "Building a Functional Puppet Workflow Part 2: Roles and Profiles"
date: 2014-02-17 14:01:37 -0800
comments: true
categories: ['puppet', 'workflow', 'roles', 'profiles', 'roles and profiles']
---

In my [first post][firstpost], I talked about writing functional component
modules. Well, I didn't really do much detailing other than pointing out
key bits of information that tend to cause problems. In this post, I'll
describe the next layer to the functional Puppet module workflow.

People usually stop once they have a library of component modules (whether
hand-written, taken from Github, or pulled from [The Forge][forge]). The idea
is that you can classify all of your nodes in `site.pp`, the Puppet
Enterprise Console, The Foreman, or with some other ENC, so why not just
declare all your classes for every node when you need them?

Because that's a lot of extra work and opportunities for fuckups.

People recognized this, so in the EARLY days of Puppet they would create node
blocks in `site.pp` and use inheritance to inherit from those blocks. This was
the right IDEA, but probably not the best PLACE for it. Eventually, 'Profiles'
were born.

The idea of 'Roles and Profiles' originally came from a piece that
[Craig Dunn wrote][craig] while he worked for the BBC, and then
[Adrien Thebo also wrote a piece][finch] that documents the same sort of pattern.
So why am I writing about it a THIRD time? Well, because I feel it's only a PIECE
of an overall puzzle. The introduction of Hiera and other awesome tools (like R10k,
which we will get to on the next post) still make Roles and Profiles VIABLE, but
they also extend upon them.

One final note before we move on - the terms 'Roles' and 'Profiles' are ENTIRELY
ARBITRARY. They're not magic reserve words in Puppet, and you can call them
whatever the hell you want. It's also been pointed out that Craig MIGHT have
misnamed them (a ROLE should be a model for an individual piece of tech, and a
PROFILE should probably be a group of roles), but, like all good Puppet Labs
employees - we suck at naming things.

## Profiles: technology-specific wrapper classes

A profile is simply a wrapper class that groups Hiera lookups and class
declarations into one functional unit. For example, if you wanted Wordpress
installed on a machine, you'd probably need to declare the apache class to
get Apache setup, declare an `apache::vhost` for the Wordpress directory,
setup a MySQL database with the appropriate classes, and so on. There are
a lot of components that go together when you setup a piece of technology,
it's not just a single class.

Because of this, a profile exists to give you a single class you can include
that will setup all the necessary bits for that piece of technology (be it
Wordpress, or Tomcat, or whatever).

Let's look at a simple profile for Wordpress:

{% codeblock lang:puppet profiles/manifests/wordpress.pp %}
class profiles::wordpress {

  ## Hiera lookups
  $site_name               = hiera('profiles::wordpress::site_name')
  $wordpress_user_password = hiera('profiles::wordpress::wordpress_user_password')
  $mysql_root_password     = hiera('profiles::wordpress::mysql_root_password')
  $wordpress_db_host       = hiera('profiles::wordpress::wordpress_db_host')
  $wordpress_db_name       = hiera('profiles::wordpress::wordpress_db_name')
  $wordpress_db_password   = hiera('profiles::wordpress::wordpress_db_password')
  $wordpress_user          = hiera('profiles::wordpress::wordpress_user')
  $wordpress_group         = hiera('profiles::wordpress::wordpress_group')
  $wordpress_docroot       = hiera('profiles::wordpress::wordpress_docroot')
  $wordpress_port          = hiera('profiles::wordpress::wordpress_port')

  ## Create user
  group { 'wordpress':
    ensure => present,
    name   => $wordpress_group,
  }
  user { 'wordpress':
    ensure   => present,
    gid      => $wordpress_group,
    password => $wordpress_user_password,
    name     => $wordpress_group,
    home     => $wordpress_docroot,
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
  apache::vhost { $::fqdn:
    port    => $wordpress_port,
    docroot => $wordpress_docroot,
  }

  ## Configure wordpress
  class { '::wordpress':
    install_dir => $wordpress_docroot,
    db_name     => $wordpress_db_name,
    db_host     => $wordpress_db_host,
    db_password => $wordpress_db_password,
  }
}
{% endcodeblock %}

### Name your profiles according to the technology they setup

Profiles are technology-specific, so you'll have one to setup wordpress, and
tomcat, and jenkins, and...well, you get the picture. You can also namespace
your profiles so that you have `profiles::ssh::server` and
`profiles::ssh::client` if you want. You can even have
`profiles::jenkins::tomcat` and `profiles::jenkins::jboss` or however you need
to namespace according to the TECHNOLOGIES you use. You don't need to include
your environment in the profile name (a la `profiles::dev::tomcat`) as the bits
of data that make the dev environment different from production should come
from **HIERA**, and thus aren't going to be different on a per-profile basis.
You CAN setup profiles according to your business unit if multiple units use
Puppet and have different setups (a la `security::profiles::tomcat` versus
`ops::profiles::tomcat`), but the GOAL of Puppet is to have one main set of
modules that every group uses (and the Hiera data being different for every
group). That's the GOAL, but I'm pragmatic enough to understand that not
everywhere is a shiny, happy 'DevOps Garden.'

### Do all Hiera lookups in the profile

You'll see that I declared variables and set their values with Hiera lookups.
The profile is the place for these lookups because the profile collects all
external data and declares all the classes you'll need. In reality, you'll
**USUALLY** only see profiles looking up parameters and declaring classes
(i.e. declaring users and groups like I did above will USUALLY be left to
component classes).

I do the Hiera lookups first to make it easy to debug from where those values
came. I don't rely on ['Automatic Parameter Lookup'][autolookup] in Puppet
3.x.x because it can be 'magic' for people who aren't aware of it (for people
new to Puppet, it's much easier to see a function call and trace back what it
does rather than experience Puppet doing something unseen and wondering what
the hell happened).

Finally, you'll notice that my Hiera lookups have **NO DEFAULT VALUES** - this
is **BY DESIGN!**  For most people, their Hiera data is **PROBABLY** located in
a separate repository as their Puppet module data. Imagine making a change to
your profile to have it lookup a bit of data from Hiera, and then imagine you
FORGOT to put that data into Hiera. What happens if you provide a default value
to Hiera? The catalog compiles, that default value gets passed down to the
component module, and gets enforced on disk. If you have good tests, you MIGHT
see that the component you configured has a bit of data that's not correct, but
what if you don't have a great post-Puppet testing workflow? Puppet will
correctly set this default value, according to Puppet everything is green and
worked just fine, but now your component is setup incorrectly. That's one of
the WORST failures - the ones that you don't catch.  Now, imagine you DON'T
provide a default value. In THIS case, Puppet will raise a compilation error
because a Hiera lookup didn't return a value. You'll catch your error before
anything gets pushed to Production and you can catch the screwup. This is
a MUCH better solution.

### Use parameterized class declarations and explicitly pass values you care about

The parameterized class declaration syntax can be dangerous. The difference
between the `include` function and the parameterized class syntax is that
the `include` function is idempotent. You can do the following in a Puppet
manifest, and Puppet doesn't raise an error:

```
include apache
include apache
include apache
```

This is because the `include` function checks to see if the class is in the
catalog. If it ISN'T, then it adds it. If it IS, then it exits cleanly. The
`include` function is your pal.

Consider THIS manifest:

```
class { 'apache': }
include apache
include apache
```

Does this work?  Yep.  The parameterized class syntax adds the class to the
catalog, the include function detects this and exits cleanly twice.  What
about THIS manifest:

```
include apache
class { 'apache': }
include apache
```

Does THIS work?  Nope!  Puppet raises a compilation error because a class was
declared more than once in a catalog.  Why?  Well, consider that Puppet is
'declarative'...all the way up until it isn't.  Puppet's PARSER reads from the
top of the file to the bottom of the file, and we have a single-pass parser
when it comes to things like setting variables and declaring classes. When
the parser hits the first include function, it adds the class to the catalog.
The parameterized class syntax, however, is a honey badger: it doesn't give
a shit. It adds a class to the catalog regardless of whether it already exists
or not. So why would we EVER use the parameterized class declaration syntax?
We need to use it because the include function doesn't allow you to pass
parameters when you declare a class.

So wait - why did I spend all this time explaining why the parameterized class
syntax is more dangerous than the include function **ONLY** to recommend its
use in profiles?  For two reasons:

* We need to use it to pass parameters to classes
* We're wrapping its use in a class that we can **IN TURN** declare with the `include` function

Yes, we can get the best of BOTH worlds, the ability to pass parameters and
the use of our pal the `include` function, with this wrapper class. We'll see
the latter usage when we come to roles, but for now let's focus on passing
parameter values.

In the first section, we set variables with Hiera lookups, now we can pass
those variables to classes we're declaring with the parameterized class syntax.
This allows the declaration of the class to be static, but the parameters we
pass to that class to change according to the Hiera hierarchy. We've explicitly
called the `hiera` function, so it makes it easier to debug, and we're explicitly
passing parameter values so we know definitively which parameters are being
passed (and thus are overriding default values) to the component module. Finally,
since our component modules do NOT use Hiera at all, we can be sure that if we're
not passing a parameter that it's getting its value from the default set in the
module's `::params` class.

Everything we do here is meant to make things easier to debug when it's 3am and
things aren't working. Any asshole can do crazy shit in Puppet, but a seasoned
sysadmin writes their code for ease of debugging during 3am pages.

### An annoying Puppet bug - top-level class declarations and profiles

[Oh, ticket 2053][2053], how terrible are you? This is one of those bug numbers
that I can remember by heart (like [8040][8040] and [86][86]). Puppet has
the ability to do 'relative namespacing', which allows you to declare a variable
called `$port` in a class called `$apache` and refer to it as `$port` instead
of fully-namespacing the variable, and thus having to call it `$apache::port`
inside the `apache` class. It's a shortcut - you can STILL refer to the variable
as `$apache::port` in the class - but it comes in handy. The PROBLEM occurs when
you create a profile, as we did above, called `profiles::wordpress` and you try
to declare a class called `wordpress`.  If you do the following inside the
`profiles::wordpress` class, what class is being declared:

```
include wordpress
```

If you think you're declaring a wordpress class from within a wordpress module
in your Puppet modulepath, you would be wrong.  Puppet ACTUALLY thinks you're
trying to declare `profiles::wordpress` because you're INSIDE the `profiles::wordpress`
class and it's doing relative namespacing (i.e. in the same way you refer to `$port`
and ACTUALLY mean `$apache::port` it thinks you're referring to `wordpress` and
ACTUALLY mean `profiles::wordpress`.

Needless to say, this causes LOTS of confusion.

The solution here is to declare a class called `::wordpress` which tells Puppet
to go to the top-level namespace and look for a module called `wordpress` which
has a top-level class called `wordpress`. It's the same reason that we refer to
Facter Fact values as `$::osfamily` instead of `$osfamily` in class definitions
(because you can declare a local variable called `$osfamily` in your class).
This is why in the profile above you see this:

```
class { '::wordpress':
  install_dir => $wordpress_docroot,
  db_name     => $wordpress_db_name,
  db_host     => $wordpress_db_host,
  db_password => $wordpress_db_password,
}
```

When you use profiles and roles, you'll need to do this namespacing trick when
declaring classes because you're frequently going to have a `profile::<sometech>`
that will declare the `<sometech>` top-level class.


## Roles: business-specific wrapper classes

How do you refer to your machines? When I ask you about that cluster over
there, do you say "Oh, you mean the machines with java 1.6, apache, mysql,
etc..."?  I didn't think so. You usually have names for them, like the
"internal compute cluster" or "app builder nodes" or "DMZ repo machines" or
whatever. These names are your Roles. Roles are just the mapping of your
machine's names to the technology that should be ON them. In the past we had
descriptive hostnames that afforded us a code for what the machine 'did' -
roles are just that mapping for Puppet.

Roles are namespaced just like profiles, but now it's up to your organization
to fill in the blanks. Some people immediately want to put environments into
the roles (a la `roles::uat::compute_cluster`), but that's usually not necessary
(as MOST LIKELY the compute cluster nodes have the SAME technology on them
when they're in dev versus when they're in prod, it's just the DATA - like
database names, VIP locations, usernames/passwords, etc - that's different.
Again, these data differences will come from Hiera, so there should be no reason
to put the environment name in your role). You still CAN put the environment
name in the role if it makes you feel better, but it'll probably be useless.

### Roles ONLY include profiles

So what exactly is in the role wrapper class? That depends on what technology
is on the node that defines that role. What I can tell you for CERTAIN is that
roles should ONLY use the `include` function and should ONLY include profiles.
What does this give us? This gives us our pal the `include` function back! You
can include the same profile 100 times if you want, and Puppet only puts it in
the catalog once. 

### Every node is classified with one role. Period.

The beautiful thing about roles and profiles is that the GOAL is that you
should be able to classify a node with a SINGLE role and THAT'S IT. This makes
classification simple and static - the node gets its role, the role includes
profiles, profiles call out to Hiera for data, that data is passed to component
modules, and away we go. Also, since classification is static, you can use
version control to see what changes were introduced to the role (i.e. what
profiles were added or removed).  In my opinion, if you need to apply more
than one role to a node, you've introduced a new role (see below).

### Roles CAN use inheritance...if you like

I've seen people implement roles a couple of different ways, and one of them
is to use inheritance to build a catalog.  For example, you can define a base
`roles` class that includes something like a base security profile (i.e.
something that EVERY node in your infrastructure should have). Moving down the
line, you COULD namespace according to function like `roles::app` for your
application server machines. The `roles::app` class could inherit from the `roles`
class (which gets the base security profile), and could then include the profiles
necessary to setup an application server. Next, you could subclass down to
`roles::app::site_foo` for an application server that supports some site in
your organization.  That class inherits from the `roles::app` class, and then
adds profiles that are specific to that site (maybe they use Jboss instead of
Tomcat, and thus that's where the differentiation occurs). This is great
because you don't have a lot of repeated use of the `include` function, but
it also makes it hard to definitively look at a specific role to see exactly
what's being declared (i.e. all the profiles). You have to weigh what you
value more: less typing or greater visibility. I will err on the side of
greater visibility (just due to that whole 3am outage thing), but it's up
to you to decide what to optimize for.

### A role similar, yet different, from another role is: a new role

EVERYBODY says to me "Gary, I have this machine that's an AWFUL LOT like
this role over here, but...it's different." My answer to them is: "Great,
that's another role."  If the thing that's different is data (i.e. which
database to connect to, or what IP address to route traffic through),
then that difference should be put in HIERA and the classification should
remain the same. If that difference is technology-specific (i.e. this server
uses JBoss instead of Tomcat) then first look and see if you can isolate
how you know this machine is different (maybe it's on a different subnet,
maybe it's at a different location, something like that). If you can figure
that out and write a Fact for it (or use similar conditional logic to determine
this logically), then you can just drop that conditional logic in your role
and let it do the heavy lifting. If, in the end, this bit of data is totally
arbitrary, then you'll need to create another role (perhaps a subclass using
the above namespacing) and assign it to your node.

The hardest thing about this setup is naming your roles. Why? Every site is
different.  It's hard for me to account for differences in your setup because
your workplace is dysfunctional (seriously).

## Review: what does this get you?

Let's walk through every level of this setup from the top to the bottom and see
what it gets you. Every node is classified to a single role, and, for the most
part, that classification isn't going to change. Now you can take all the extra
work off your classifier tool and put it back into the manifests (that are
subject to version control, so you can `git blame` to your heart's content and
see who last changed the role/profile). Each role is going to include one or
more profile, which gives us the added idempotent protection of the include
function (of course, if profiles have collisions with classes you'll have to
resolve those. Say one or more profiles tries to include an apache class -
simply break that component out into a separate profile, extract the parameters
from Hiera, and include that profile at a higher level). Each profile is going
to do Hiera lookups which should give you the ability to provide different data
for different host types (i.e. different data on a per-environment level, or
however you lay out your Hiera hierarchy), and that data will be passed
directly to class that is declared. Finally, each component module will
accept parameters as variables internal to that module, default
parameters/variables to sane values in the `::params` class, and use those
variables when declaring each resource throughtout its classes.

* Roles abstract profiles
* Profiles abstract component modules
* Hiera abstracts configuration data
* Component modules abstract resources
* Resources abstract the underlying OS implementation

## Choose your level of comfortability

The roles and profiles pattern also buys you something else - the ability for
less-skilled and more-skilled Puppet users to work with the same codebase.
Let's say you use some GUI classifier (like the Puppet Enterprise Console),
someone who's less skilled at Puppet looks and sees that a node is classified
with a certain role, so they open the role file and see something like this:

```
include profiles::wordpress
include profiles::tomcat
include profiles::git::repo_server
```

That's pretty legible, right? Someone who doesn't regularly use Puppet can
probably make a good guess as to what's on the machine. Need more information?
Open one of the profiles and look specifically at the classes that are being
declared. Need to know the data being passed? Jump into Hiera. Need to know
more information? Dig into each component module and see what's going on there.

When you have everything abstracted correctly, you can have developers
providing data (like build versions) to Hiera, junior admins grouping nodes for
classification, more senior folk updating profiles, and your best Puppet people
creating/updating component modules and building plugins like custom
facts/functions/whatever. 

## Great! Now go and refactor...

If you've used Puppet for more than a month, you're probably familiar with the
"Oh shit, I should have done it THAT way...let me refactor this" game. I know,
it sucks, and we at Puppet Labs haven't been shy of incorporating something that
we feel will help people out (but will also require some refactoring). This
pattern, though, has been in use by the Professional Services team at Puppet Labs
for over a year without modification. I've used this on sites GREAT and small,
and every site with which I've consulted and implemented this pattern has been
able to both understand its power and derive real value within a week. If you're
contemplating a refactor, you can't go wrong with Roles and Profiles (or
whatever names you decide to use).

  
[puppetlabs]: http://www.puppetlabs.com
[rimodule]: http://www.devco.net/archives/2012/12/13/simple-puppet-module-structure-redux.php
[docsmodule]: http://docs.puppetlabs.com/guides/module_guides/bgtm.html
[almodule]: http://www.confreaks.com/videos/1651-puppetconf2012-puppet-modules-for-fun-and-profit
[hierablog]: http://garylarizza.com/blog/2013/12/08/when-to-hiera/
[forge]: http://forge.puppetlabs.com
[craig]: http://www.craigdunn.org/2012/05/239/
[finch]: http://sysadvent.blogspot.com/2012/12/day-13-configuration-management-as-legos.html
[firstpost]: http://www.garylarizza.com/blog/2014/02/17/puppet-workflow-part-1/
[autolookup]: http://docs.puppetlabs.com/hiera/1/puppet.html#automatic-parameter-lookup
[2053]: http://projects.puppetlabs.com/issues/2053
[86]: http://projects.puppetlabs.com/issues/86
[8040]: http://projects.puppetlabs.com/issues/8040 
