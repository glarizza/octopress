---
layout: post
title: "Fun with Puppet Providers - Part 1 of Whatever"
date: 2013-11-25 20:47
comments: true
categories: ['os x', 'puppet', 'types', 'providers', 'proxies']
---

I don't know why I write blog posts - everybody in open-source software knows
that the code *IS* the documentation. If you've ever tried to write a Puppet
type/provider, you know this fact better than ANYONE. To this day, when someone
asks me for the definitive source on this activity I usually refer them first
to [Nan Liu and Dan Bode's awesome Types and Providers
book](http://www.amazon.com/Puppet-Types-Providers-Dan-Bode/dp/1449339328)
(which REALLY is a fair bit of quality information), and THEN to the source
code for Puppet. Everything else falls in-between those sources (sadly).

As someone who truly came from knowing absolute fuckall about Ruby and only
marginally more than that about Puppet, I've walked through the valley of the
shadow of self.instances and have survived to tell the tale. That's what this
post is about - hopefully some GOOD information if you want to start writing
your own Puppet type and provider. I also wrote this because this knowledge
has been passed down from Puppet employee to Puppet employee, and I wanted
to break the priesthood being held on type and provider magic. If you don't
hear from me after tomorrow, well, then you know what happened...

## Because 20 execs in a defined type...

What would drive someone to write a custom type and provider for Puppet anyhow?
Afterall, you can do ANYTHING IMAGINABLE in the Puppet DSL\*! After drawing
back my sarcasm a bit, let me explain where the Puppet DSL tends to fall over
and the idea of a custom type and provider starts becoming more than just an
incredibly vivid dream:

- You have more than a couple of exec statements in a single class/defined type that have multiple conditional
properties like 'onlyif' and/or 'unless'.
- You need to use pure Ruby to manipulate data and parse it through a system binary
- Your defined type has more conditional logic than your pre-nuptual agreement
- Any combination of similar arguments related to the above

If the above sounds familiar to you, then you're probably ready to build your own custom Puppet type and provider.
Do note that custom types and providers are written in Ruby and not the Puppet DSL. This can initially feel
very scary, but get over it (there are much scarier things coming).

*\* Just because you can doesn't mean you don't, in fact, suck.*

## I'm not your Type

This blog post is going to focus on types and type-interaction, while later
posts will focus on providers and ultimately dirty provider tricks to win
friends and influence others.  Type and provider interaction can be totally
daunting for newcomers, let ALONE just naming files correctly due to Puppet's
predictable (note: anytime I write the word "predictable", just substitute the
phrase "annoying pain in the ass") naming pattern.  Let's break it down a bit
for you - somebody que Dre...

(NOTE: I'm going to ASSUME you understand the fundamentals of a Puppet run already. If you're pretty hazy
on that concept, checkout [docs.puppetlabs.com](http://docs.puppetlabs.com) for more information)

## Types are concerned about your looks

The type file defines all the *properties* and *parameters* that can be used by your new custom resource.
Think of the type file like the opening stanza to a new Puppet class - we're describing all the tweakable
knobs and buttons to the new thing we're creating. The type file also gives you some added validation
abilities, which is very handy.

It's important to understand that there is a BIG difference between a 'property' and a 'parameter' with regard
to a type (even though they're both assigned values identically in a resource declaration).  Think of it this
way: a property is something that can be inspected and changed by Puppet, while a parameter is just helper data
that Puppet uses to do its job.  A property would be something like a file's mode.  You can inspect a file and
determine its mode, and you can even CHANGE a file's mode on disk. The file resource type also has a parameter
called 'backup'.  Its sole job is to tell Puppet whether to backup the file to the filebucket before making
changes. This data is useful for Puppet during a run, but you can't inspect a file on disk and know definitively
whether Puppet is going to back it up or not (and it goes without saying that if you can't determine this aspect
about a file on disk just by inspecting it, than you also can't CHANGE this aspect about a file on disk either).
You'll see later where the property/parameter distinction becomes very important.

Recently I built a type modeling the setting of proxy data for network interfaces on OS X, so we'll use that as
a demonstration of a type.  It looks like the following:

{% codeblock lib/puppet/type/mac_web_proxy.rb lang:ruby %}
Puppet::Type.newtype(:mac_web_proxy) do
  desc "Puppet type that models a network interface on OS X"

  ensurable

  newparam(:name, :namevar => true) do
    desc "Interface name - currently must be 'friendly' name (e.g. Ethernet)"
    munge do |value|
      value.downcase
    end
    def insync?(is)
      is.downcase == should.downcase
    end
  end

  newproperty(:proxy_server) do
    desc "Proxy Server setting for the interface"
  end

  newparam(:authenticated_username) do
    desc "Username for proxy authentication"
  end

  newparam(:authenticated_password) do
    desc "Password for proxy authentication"
  end

  newproperty(:proxy_authenticated) do
    desc "Proxy Server setting for the interface"
    newvalues(:true, :false)
  end

  newproperty(:proxy_port) do
    desc "Proxy Server setting for the interface"
    newvalues(/^\d+$/)
  end
end
{% endcodeblock %}

First note the type file's path in the grey titlebar of the graphic: `lib/puppet/type/mac_web_proxy.rb`
This path is relative to the module that you're building, and it's VERY important that it be named
EXACTLY this way to appease Puppet's predictable naming pattern.  The name of the file directly correllates
to the name of the type listed in the `Puppet::Type.newtype()` method.

Next, let's look at a sample parameter declaration - for starters, let's look at the 'authenticated_password'
parameter declaration on line 24 of the above type.  The `newparam()` method is called and the lone argument
passed is the symbolized name of our parameter (i.e. it's prepended with a colon).  This parameter
provides the password to use when setting up an authenticated web proxy on OS X.
It's a parameter because as far as I know, there's no way for me to query the system for this password
(it's obfuscated in the GUI and I'm not entirely certain where it's stored on-disk).
If there were a way for us to query this value from the system, then we could turn it
into a property (since we could both 'GET' as well as 'SET' the value). As of right
now, it exists as helper data for when I need to setup an authenticated proxy.

Having seen a parameter, let's look at the 'proxy_server' property that's declared on
line 16 of the type file above. We're able to both query the system for this value,
as well as change/set the value by using the `networksetup` binary, so it's able to
be 'synchronized' (according to Puppet). Because of this, it must be a property.


## Just enough validation

The second major function of the type file is to provide methods to validate property
and parameter data that is being passed. There are two methods to validate this data,
and one method that allows you to massage the data into an acceptable format (which
is called 'munging').

### validate()

The first method, named 'validate', is widely believed to be the only successfully-named
method in the entire Puppet codebase. Validate accepts a block and allows you to perform
free-form validation in any way you prefer.  For example:

{% codeblock lib/puppet/type/user.rb lang:ruby %}
validate do |value|
  raise ArgumentError, "Passwords cannot include ':'" if value.is_a?(String) and value.include?(":")
end
{% endcodeblock %}

This example, pulled straight from the Puppet codebase, will raise an error if a
password contains a colon. In this case, we're looking for a specific exception
and are raising errors accordingly.

### newvalues()

The second method, named 'newvalues', accepts a regex that property/parameter values
need to match (if you're one of the 8 people in the world that speak regex fluently), 
or a list of acceptable values. From the example above:

{% codeblock lib/puppet/type/mac_web_proxy.rb lang:ruby %}
  newproperty(:proxy_authenticated) do
    desc "Proxy Server setting for the interface"
    newvalues(:true, :false)
  end

  newproperty(:proxy_port) do
    desc "Proxy Server setting for the interface"
    newvalues(/^\d+$/)
  end
{% endcodeblock %}

### munge()

The final method, named 'munge' accepts a block like `newvalues` but allows you to
convert an unacceptable value into an acceptable value. Again, this is from the example above:

{% codeblock lib/puppet/type/mac_web_proxy.rb lang:ruby %}
munge do |value|
  value.downcase
end
{% endcodeblock %}

In this case, we want to ensure that the parameter value is lower case. It's not
necessary to throw an error, but rather it's acceptable to 'munge' the value to
something that is more acceptable without alerting the user.

## Important type considerations

You could write half a book just on how types work (and, again, check out the book
referenced above which DOES just that), but there are a couple of final considerations
that will prove helpful when developing your type.

### Defaulting values

The `defaultto` method provides a default value should the user not provide one for
your property/parameter. It's a pretty simple construct, but it's important to
remember when you write spec tests for your type (which you ARE doing, right?) that
there will ALWAYS be values for properties/parameters that utilize `defaultto`. Here's a quick example:

{% codeblock Defaultto example lang:ruby %}
newparam(:enable_lacp) do
  defaultto :true
  newvalues(:true, :false)
end
{% endcodeblock %}

### Ensurable types

A resource is considered 'ensurable' when its presence can be verified (i.e. it
exists on the system), it can be created when it doesn't exist and it SHOULD, and
it can be destroyed when it exists and it SHOULDN'T. The simplest way to tell
Puppet that a resource type is ensurable is to call the `ensurable` method within
the body of the type (i.e. outside of any property/parameter declarations). Doing
this will automatically create an 'ensure' property that accepts values of 'absent'
and 'present' that are automatically wired to the 'exists?', 'create' and 'destroy'
methods of the provider (something I'll write about in the next post). Optionally,
you can choose to pass a block to the `ensurable` method and define acceptable
property values as well as the methods of the provider that are to be called. That
would look something like this:

{% codeblock lib/puppet/type/package.rb lang:ruby %}
ensurable do
  newvalue(:present) do
    provider.install
  end

  newvalue(:absent) do
    provider.uninstall
  end

  newvalue(:purged) do
    provider.purge
  end

  newvalue(:held) do
    provider.hold
  end
end
{% endcodeblock %}

This means that instead of calling the `create` method to create a new resource that
SHOULD exist (but doesn't), Puppet is going to call the `install` method. Conversely,
it will call the `uninstall` method to destroy a resource based on this type. The
ensure property will also accept values of 'purged' and 'held' which will be wired up
to the `purge` and `hold` methods respectively.

### Namevars are unique little snowflakes

Puppet has a [concept known as the 'namevar' for a resource.](http://docs.puppetlabs.com/puppet/2.7/reference/lang_resources.html#namenamevar)
If you're hazy about the concept check out the documentation, but basically it's the parameter
that describes the form of uniqueness for a resource type on the system. For the package resource
type, the 'name' parameter is the namevar because the way you tell one package from another is
its name. For the file resource, it's the 'path' parameter, because you can differentiate unique
files from each other according to their path (and not necessarily their filename, since filenames
don't have to be unique on systems).

When designing a type, it's important to consider WHICH parameter will be the namevar (i.e. how
can you tell unique resources from one another). To make a parameter the namevar, you simply
set the `:namevar` attribute to `:true` like below:

{% codeblock lang:ruby %}
newparam(:name, :namevar => :true) do
  # Type declaration attributes here...
end
{% endcodeblock %}

### Handling array values

Nearly every property/parameter value that is declared for a resource is 'stringified', or
cast to a string. Sometimes, however, it's necessary to accept an array of elements as the
value for a property/parameter. To do this, you have to explicitly tell Puppet that you'll
be passing an array by setting the `:array_matching` attribute to `:all` (if you don't set
this attribute, it defaults to `:first`, which means that if you pass an array as a value
for a property/parameter, Puppet will only accept the FIRST element in that array).

{% codeblock lang:ruby %}
newproperty(:domains, :array_matching => :all) do
  # Type declaration attributes here... 
end
{% endcodeblock %}

If you set `:array_matching` to `:all`, EVERY value passed for that parameter/property will
be cast to an array (which means if you pass a value of 'foo', you'll get an array with a
single element - the string of 'foo').

### Documenting your property/parameter

It's a best-practice to document the purpose of your property or parameter declaration, and
this can be done by passing a string to the `desc` method within the body of the property/parameter
declaration.

{% codeblock lang:ruby %}
newproperty(:domains, :array_matching => :all) do
  desc "Domains which should bypass the proxy"
# Type declaration attributes here...
end
{% endcodeblock %}

### Synchronization tricks

Puppet uses a method called `insync?` to determine whether a property value is synchronized (i.e.
if Puppet needs to change its value, or it's set appropriately). You usually have no need to change
the behavior of this method since most of the properties you create for a type will have string
values (and the `==` operator does a good job of checking string equality). For structured data
types like arrays and hashes, however, that can be a bit trickier. Arrays, for example, are
ordered construct - they have a definitive idea of what the first element and the last element
of the array are. Sometimes you WANT to ensure that values are in a very specific order, and
sometimes you don't necessarily care about the ORDER that values for a property are set - you
just want to make sure that all of them are set.

If the latter cases sounds like what you need, then you'll need to override the behavior of the
`insync?` method. Take a look at the below example:

{% codeblock lang:ruby %}
newproperty(:domains, :array_matching => :all) do
  desc "Domains which should bypass the proxy"
  def insync?(is)
    is.sort == should.sort
  end
end
{% endcodeblock %}

In this case, I've overridden the `insync?` method to first sort the 'is' value (or, the value that
was discovered by Puppet on the target node) and compare it with the sorted 'should' value (or,
the value that was specified in the Puppet manifest when the catalog was compiled by the Puppet
master). You can do WHATEVER you want in here as long as `insync?` returns either a true or a
false value.  If `insync?` returns true, then Puppet determines that everything is in sync and
no changes are necessary, whereas if it returns false then Puppet will trigger a change.

## And this was the EASY part!

Wow this went longer than I expected... and types are usually the 'easier' bit
since you're only describing the format to be used by the Puppet admin in
manifests. There are some hacky type tricks that I've not yet covered (i.e.
features, 'inheritance', and other meta-bullshit), but those will be saved for
a final 'dirty tips and tricks' post.  In the next section, I'll touch on
providers (which is where all interaction with the system takes place), so
stay tuned for more brain-dumping-goodness...
