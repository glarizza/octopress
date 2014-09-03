---
layout: post
title: "Seriously, what is this provider doing?"
date: 2013-12-15 11:44:17 -0500
comments: true
categories: ['puppet', 'ruby', 'types', 'providers', 'dirty black magic']
---

Clarke's third law states: "Any sufficiently advanced technology is
indistinguishable from magic." In the case of Ruby and Puppet provider
interaction, I'm inclined to believe it. If you want proof, take a look at some
of the native Puppet types - no amount of 'Expecto Patronum' will free you from
the Ruby metaprogramming dementors that hover around
`lib/puppet/provider/exec`-land.

In my [first post tackling Puppet types and providers][firstpost], I introduced
the concept of Puppet types and the utility they provide. [In the second post,][secondpost]
I brought you to the great plain of Puppet providers and introduced the core
methods necessary for creating a very basic Puppet provider with a single property
(HINT: if you've not read either of those posts, or you've never dealt with basic
types and providers, you might want to stop here and read up a bit on the topics).
The problems with a provider like the one created in that post were:

* `puppet resource` support wasn't implemented, so you couldn't query for existing instances of the type on the system (and their corresponding values)
* The getter method would be called for EVERY instance of the type on the system, which would mean shelling-out multiple times during a run
* Ditto for the setter method (if changes to multiple instances of the type were necessary)
* That type was VERY basic (i.e. ensurable with a single property)

Unfortunately, when most of us have the need of a Puppet type and provider, we
usually require multiple properties and reasonably complex system interaction. When
it comes to creating both a getter and a setter method for every property (including
the potential performance hit that could come from shelling-out many times during
a Puppet run), ain't nobody got time for that. And finally, `puppet resource` is a
REALLY handy tool for querying the current state of your resources on a system.
These problems all have solutions, but up until recently there was just one more
problem:

**Good luck finding documentation for those solutions.**

NOTE: The [Puppet Types and Providers book written by Nan and Dan][puppetbook]
is a great resource that provides a bit of a deeper dive than I'll be doing in this
post - DO check it out if you want to know more

## Something, something, `puppet resource`

The `puppet resource` command (or `ralsh`, as it used to be known), is a very
handy command for querying a system and returning the current state of resources
for a specific Puppet type.  Try it out if you never have (note that the following
is being run on CentOS 6.4):

    [root@linux ~]# puppet resource user
    user { 'abrt':
      ensure           => 'present',
      gid              => '173',
      home             => '/etc/abrt',
      password         => '!!',
      password_max_age => '-1',
      password_min_age => '-1',
      shell            => '/sbin/nologin',
      uid              => '173',
    }
    user { 'adm':
      ensure           => 'present',
      comment          => 'adm',
      gid              => '4',
      groups           => ['sys', 'adm'],
      home             => '/var/adm',
      password         => '*',
      password_max_age => '99999',
      password_min_age => '0',
      shell            => '/sbin/nologin',
      uid              => '3',
    }
    < ... and more users below ... >

The `puppet resource` command returns a list of all users on the system
and their current property values (note you can only see the password
hash if you're running Puppet with sufficient privileges). You can even
query `puppet resource` for the values of a specific resource:

    [root@gary ~]# puppet resource user glarizza
    user { 'glarizza':
      ensure           => 'present',
      gid              => '502',
      home             => '/home/glarizza',
      password         => '$1$hsUuCygh$kgLKG5epuRaXHMX5KmxrL1',
      password_max_age => '99999',
      password_min_age => '0',
      shell            => '/bin/bash',
      uid              => '502',
    }

`puppet resource` seems magical, and you might think that if you create a
custom type and sync it to your machine then `puppet resource` will automatically
work for you.

**And you would be wrong.**

`puppet resource` will only work if you've implemented a special method in your
provider called `self.instances`.

## `self.instances`

The `self.instances` method is pretty sparsely documented, so let's go straight
to the source...code, that is:

{% codeblock lang:ruby lib/puppet/provider.rb %}
  # Returns a list of system resources (entities) this provider may/can manage.
  # This is a query mechanism that lists entities that the provider may manage on a given system. It is
  # is directly used in query services, but is also the foundation for other services; prefetching, and
  # purging.
  #
  # As an example, a package provider lists all installed packages. (In contrast, the File provider does
  # not list all files on the file-system as that would make execution incredibly slow). An implementation
  # of this method should be made if it is possible to quickly (with a single system call) provide all
  # instances.
  #
  # An implementation of this method should only cache the values of properties
  # if they are discovered as part of the process for finding existing resources.
  # Resource properties that require additional commands (than those used to determine existence/identity)
  # should be implemented in their respective getter method. (This is important from a performance perspective;
  # it may be expensive to compute, as well as wasteful as all discovered resources may perhaps not be managed).
  #
  # An implementation may return an empty list (naturally with the effect that it is not possible to query
  # for manageable entities).
  #
  # By implementing this method, it is possible to use the `resources´ resource type to specify purging
  # of all non managed entities.
  #
  # @note The returned instances are instance of some subclass of Provider, not resources.
  # @return [Array<Puppet::Provider>] a list of providers referencing the system entities
  # @abstract this method must be implemented by a subclass and this super method should never be called as it raises an exception.
  # @raise [Puppet::DevError] Error indicating that the method should have been implemented by subclass.
  # @see prefetch
  def self.instances
    raise Puppet::DevError, "Provider #{self.name} has not defined the 'instances' class method"
  end
{% endcodeblock %}

You'll find that method around lines 348 - 377 of the `lib/puppet/provider.rb`
file in Puppet's source code (as of this writing, which is a Friday...
on a flight from DC to Seattle). To summarize, implementing `self.instances`
in your provider means that you need to return an array of provider instances
that have been discovered on the current system and all the current property
values (we call these values the 'is' values for the properties, since each
value IS the current value of the property on the system). It's recommended to only
implement `self.instances` if you can gather all resource property values in a reasonably
'cheap' manner (i.e. a single system call, read from a single file, or some
similar low-IO means). Implementing `self.instances` not only gives you the
ability to run `puppet resource` (which also affords you a quick-and-dirty
way of testing your provider without creating unit tests by simply running
`puppet resource` in debug mode and checking the output), but it also allows
the ['resources' resource][resourcesresource] to work its magic (If you've
never heard of the 'resources' resource, [check this link][resourcesresource]
for more information on this terribly/awesomely named resource type).

### An important note about scope and `self.instances`

The `self.instances` method is a method of the PROVIDER, which is why it is
prefixed with `self.`  Even though it may be located in the provider file
itself, and even though it sits among other methods like `create`, `exists?`,
and `destroy` (which are methods of the INSTANCE of the provider), it does
NOT have the ability to directly access or call those methods. It DOES have
the ability to access other methods of the provider directly (i.e. other
methods prefixed with `self.`).  This means that if you were to define a
method like:

{% codeblock lang:ruby %}
def self.proxy_type
  'web'
end
{% endcodeblock %}

You could access that directly from `self.instances` by simply calling it:

{% codeblock lang:ruby %}
type_of_proxy = proxy_type()
{% endcodeblock %}

Let's say you had a method of the INSTANCE of the provider, like so:

{% codeblock lang:ruby %}
def system_type
  'OS X'
end
{% endcodeblock %}

You COULD NOT access this method from `self.instances` directly (there are
always hacky ways around EVERYTHING in Ruby, sure, but there is no easy/straightforward
way to access this method).

**And here's where it gets confusing...**

Methods of the INSTANCE of the provider CAN access provider methods directly.
Given our previous example, what if the `system_type` method wanted to access
`self.proxy_type` for some reason?  It could be done like so:

{% codeblock lang:ruby %}
def system_type
  type_of_proxy = self.class.proxy_type()
  'OS X'
end
{% endcodeblock %}

A method of the instance of the provider can access provider methods by simply
calling the `class` method on itself (which returns the provider object). This
is a one-way street for method creation that needs to be heeded when designing
your provider.

## Building a provider that uses `self.instances` (or: more Mac problems)

In the previous two posts on types/providers, I created a type and provider
for managing bypass domains for network proxies on OS X. For this post, let's
create a provider for actually MANAGING the proxy settings for a given
network interface.  Here's a quick type for managing a web proxy on a network
interface on OS X:

{% codeblock lang:ruby puppet-mac_proxy/lib/puppet/type/mac_web_proxy.rb %}
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

This type has three properties, is ensurable, and a namevar called 'name'. As
for the provider, let's start with `self.instances` and get the web proxy
values for all interfaces.  To do that we're going to need to know how to get
a list of all network interfaces, and also how to get the current proxy state
for every interface. Fortunately, both of those tasks are accomplished with the
`networksetup` binary:

    ▷ networksetup -listallnetworkservices
    An asterisk (*) denotes that a network service is disabled.
    Bluetooth DUN
    Display Ethernet
    Ethernet
    FireWire
    Wi-Fi
    iPhone USB
    Bluetooth PAN

    ▷ networksetup -getwebproxy Ethernet
    Enabled: No
    Server: proxy.corp.net
    Port: 1234
    Authenticated Proxy Enabled: 0

Cool, so one binary will do both tasks and they're REASONABLY low-cost to run.

### Helper methods

To keep things separated and easier to test, let's create separate helper methods
for each task. Since these methods are going to be called by `self.instances`,
they will be provider methods.
  
The first method will simply return an array of network interfaces:

{% codeblock lang:ruby %}
def self.get_list_of_interfaces
  interfaces = networksetup('-listallnetworkservices').split("\n")
  interfaces.shift
  interfaces.sort
end
{% endcodeblock %}

Remember from above that the `networksetup -listallnetworkservices` command
returns an info line before each interface, so this code strips that line
off and returns a sorted list of interfaces based on a one-line-per-interface
assumption.

The next method we need will accept a network interface name as an argument,
will run the `networksetup -getwebproxy (interface)` command, and will use
its output to return all the current property values (including the ensure
value) for every instance of the type on the system (i.e. every interface's
proxy settings and whether the proxy is enabled, which means the resource
is ensured as 'present', or disabled, which means the resource is ensured
as 'absent'.  

{% codeblock lang:ruby %}
def self.get_proxy_properties(int)
  interface_properties = {}

  begin
    output = networksetup(['-getwebproxy', int])
  rescue Puppet::ExecutionFailure => e
    raise Puppet::Error, "#mac_web_proxy tried to run `networksetup -getwebproxy #{int}` and the command returned non-zero. Failing here..."
  end

  output_array = output.split("\n")
  output_array.each do |line|
    line_values = line.split(':')
    line_values.last.strip!
    case line_values.first
    when 'Enabled'
      interface_properties[:ensure] = line_values.last == 'No' ? :absent : :present
    when 'Server'
      interface_properties[:proxy_server] = line_values.last.empty? ? nil : line_values.last
    when 'Port'
      interface_properties[:proxy_port] = line_values.last == '0' ? nil : line_values.last
    when 'Authenticated Proxy Enabled'
      interface_properties[:proxy_authenticated] = line_values.last == '0' ? nil : line_values.last
    end
  end

  interface_properties[:provider] = :ruby
  interface_properties[:name]     = int.downcase
  interface_properties
end
{% endcodeblock %}

A couple of notes on the method itself - first, the `networksetup` command must
exit zero on success or non-zero on failure (which it does). If ever the `networksetup`
command were to return non-zero, we're raising our own Puppet::Error, documenting what
happened, and bailing out.

This method is going to return a hash of properties and values that is going
to be used by `self.instances` - so the case statement needs to account for that.
HOWEVER you populate that hash is up to you (in my case, I'm checking for specific
output that `networksetup` returns), but make sure that the hash has a value for
the `:ensure` key at the VERY least.

### Assembling `self.instances`

Once the helper provider methods have been defined, `self.instances` becomes
reasonably simple:

{% codeblock lang:ruby %}
def self.instances
  get_list_of_interfaces.collect do |int|
    proxy_properties = get_proxy_properties(int)
    new(proxy_properties)
  end
end
{% endcodeblock %}

Remember that `self.instances` must return an array of provider instances,
and each one of these instances must include the `namevar` and ensure value
at the very least. Since `self.get_proxy_properties` returns a hash containing
all the property 'is' values for a resource, declaring a new provider instance
is as easy as calling the `new()` method on the return value of
`self.get_proxy_properties` for every network interface. In the end, the
return value of the `collect` method on `get_list_of_interfaces` will be an
array of provider instances.

## Existance, `@property_hash`, and more magical methods

Even though we have assembled a functional `self.instances` method, we don't have
complete implementation that will work with `puppet resource`.  The problem is
that Puppet can't yet determine the existance of a resource (even though the
resource's `ensure` value has been set by `self.instances`). If you were to
execute the code with `puppet resource mac_web_proxy`, you would get the error:

    Error: Could not run: No ability to determine if mac_web_proxy exists

To satisfy Puppet, we need to implement an `exists?()` method for the instance
of the provider. Fortunately, we don't need to re-implement any existing logic
and can instead use `@property_hash`

### A `@property_hash` is born...

I've omitted one last thing that is borne out of `self.instances`, and that's
the `@property_hash` instance variable. `@property_hash` is populated by
`self.instances` as an instance variable that's available to methods of the
INSTANCE of the provider (i.e. methods that ARE NOT prefixed with `self.`)
containing all the 'is' values for a resource. Do you need to get the 'is'
value for a property?  Just use `@property_hash[:property_name]`. Since the
`exists?` method is a method of the instance of the provider, and it's
essentially the same thing as the `ensure` value for a resource, let's implement
`exists?` by doing a check on the ensure value from the `@property_hash`
variable:

{% codeblock lang:ruby %}
def exists?
  @property_hash[:ensure] == :present
end
{% endcodeblock %}

Perfect, now `exists?` will return true or false accordingly and Puppet will be
satisfied.

### Getter methods - the slow way

Puppet may be happy that you have an `exists?` method, but `puppet resource`
won't successfully run until you have a method that returns an 'is' value for
every property of the type (i.e. the `proxy_server`, `proxy_authenticated`,
and `proxy_port` attributes for the mac_web_proxy type). These 'is value methods' are
called 'getter' methods: they're methods of the instance of the provider, and
are named exactly the same as the properties they represent.

You SHOULD be thinking: "Hey, we already have `@property_hash`, why can't we
just use it again? We can, and you COULD implement all the getter methods
like so:

{% codeblock lang:ruby %}
def proxy_server
  @property_hash[:proxy_server]
end
{% endcodeblock %}

If you did that, you would be TECHNICALLY correct, but it would seem to be a
waste of lines in a provider (especially if you have many properties).

### Getter methods - the quicker 'method'

Because uncle Luke hated excess lines of code, he made available a method called
`mk_resource_methods` which works very similarly to Ruby's `attr_accessor`
method. Adding `mk_resource_methods` to your provider will AUTOMATICALLY
create getter methods that pull values out of `@property_hash` in the similar
way that I just demonstrated (it will also create SETTER methods too, but we'll
look at those later). Long story short - don't make getter/setter methods if
you're using self.instances - just implement `mk_resource_methods`.

## JUST enough for `puppet resource`

Putting everything that we've learned up until now, we should have a provider
that looks like this:

{% codeblock lang:ruby lib/puppet/provider/mac_web_proxy/ruby.rb %}
Puppet::Type.type(:mac_web_proxy).provide(:ruby) do
  commands :networksetup => 'networksetup'

  mk_resource_methods

  def self.get_list_of_interfaces
    interfaces = networksetup('-listallnetworkservices').split("\n")
    interfaces.shift
    interfaces.sort
  end

  def self.get_proxy_properties(int)
    interface_properties = {}

    begin
      output = networksetup(['-getwebproxy', int])
    rescue Puppet::ExecutionFailure => e
      Puppet.debug "#get_proxy_properties had an error -> #{e.inspect}"
      return {}
    end

    output_array = output.split("\n")
    output_array.each do |line|
      line_values = line.split(':')
      line_values.last.strip!
      case line_values.first
      when 'Enabled'
        interface_properties[:ensure] = line_values.last == 'No' ? :absent : :present
      when 'Server'
        interface_properties[:proxy_server] = line_values.last.empty? ? nil : line_values.last
      when 'Port'
        interface_properties[:proxy_port] = line_values.last == '0' ? nil : line_values.last
      when 'Authenticated Proxy Enabled'
        interface_properties[:proxy_authenticated] = line_values.last == '0' ? nil : line_values.last
      end
    end

    interface_properties[:provider] = :ruby
    interface_properties[:name]     = int.downcase
    Puppet.debug "Interface properties: #{interface_properties.inspect}"
    interface_properties
  end

  def self.instances
    get_list_of_interfaces.collect do |int|
      proxy_properties = get_proxy_properties(int)
      new(proxy_properties)
    end
  end

  def exists?
    @property_hash[:ensure] == :present
  end
end
{% endcodeblock %}

Here's a tree of the module I've assembled on my machine:

    └(~/src/puppet-mac_web_proxy)▷ tree .
    .
    └── lib
       └── puppet
           ├── provider
           │   └── mac_web_proxy
           │       └── ruby.rb
           └── type
               └── mac_web_proxy.rb

To test out `puppet resource`, we need to make Puppet aware of our new custom
module. To do that, let's set the `$RUBYLIB` environmental variable. `$RUBYLIB`
is queried by Puppet and is added to its load path when looking for additional
Puppet plugins.  You will need to set `$RUBYLIB` to the path of the `lib` directory
in the custom module that you've assembled.  Because my custom module is located
in `~/src/puppet-mac_web_proxy`, I'm going to set `$RUBYLIB` like so:

    export RUBYLIB=~/src/puppet-mac_web_proxy/lib

You can execute that command from the command line, or set it in your
`~/.{bash,zsh}rc` and `source` that file.

Finally, with all the files in place and `$RUBYLIB` set, it's time to officially
run `puppet resource` (I'm going to do it in `--debug` mode to see the debug output
that I've written into the code):

    └(~/src/blogtests)▷ envpuppet puppet resource mac_web_proxy --debug
    Debug: Executing '/usr/sbin/networksetup -listallnetworkservices'
    Debug: Executing '/usr/sbin/networksetup -getwebproxy Bluetooth DUN'
    Debug: Interface properties: {:ensure=>:absent, :proxy_server=>nil, :proxy_port=>nil, :proxy_authenticated=>nil, :provider=>:ruby, :name=>"bluetooth dun"}
    Debug: Executing '/usr/sbin/networksetup -getwebproxy Bluetooth PAN'
    Debug: Interface properties: {:ensure=>:absent, :proxy_server=>nil, :proxy_port=>nil, :proxy_authenticated=>nil, :provider=>:ruby, :name=>"bluetooth pan"}
    Debug: Executing '/usr/sbin/networksetup -getwebproxy Display Ethernet'
    Debug: Interface properties: {:ensure=>:absent, :proxy_server=>"foo.bar.baz", :proxy_port=>"80", :proxy_authenticated=>nil, :provider=>:ruby, :name=>"display ethernet"}
    Debug: Executing '/usr/sbin/networksetup -getwebproxy Ethernet'
    Debug: Interface properties: {:ensure=>:absent, :proxy_server=>"proxy.corp.net", :proxy_port=>"1234", :proxy_authenticated=>nil, :provider=>:ruby, :name=>"ethernet"}
    Debug: Executing '/usr/sbin/networksetup -getwebproxy FireWire'
    Debug: Interface properties: {:ensure=>:present, :proxy_server=>"stuff.bar.blat", :proxy_port=>"8190", :proxy_authenticated=>nil, :provider=>:ruby, :name=>"firewire"}
    Debug: Executing '/usr/sbin/networksetup -getwebproxy Wi-Fi'
    Debug: Interface properties: {:ensure=>:absent, :proxy_server=>nil, :proxy_port=>nil, :proxy_authenticated=>nil, :provider=>:ruby, :name=>"wi-fi"}
    Debug: Executing '/usr/sbin/networksetup -getwebproxy iPhone USB'
    Debug: Interface properties: {:ensure=>:absent, :proxy_server=>nil, :proxy_port=>nil, :proxy_authenticated=>nil, :provider=>:ruby, :name=>"iphone usb"}
    mac_web_proxy { 'bluetooth dun':
      ensure => 'absent',
    }
    mac_web_proxy { 'bluetooth pan':
      ensure => 'absent',
    }
    mac_web_proxy { 'display ethernet':
      ensure => 'absent',
    }
    mac_web_proxy { 'ethernet':
      ensure => 'absent',
    }
    mac_web_proxy { 'firewire':
      ensure       => 'present',
      proxy_port   => '8190',
      proxy_server => 'stuff.bar.blat',
    }
    mac_web_proxy { 'iphone usb':
      ensure => 'absent',
    }
    mac_web_proxy { 'wi-fi':
      ensure => 'absent',
    }

Note that you will only see 'is' values if you have a proxy set on any of your
network interfaces (obviously, if you've not setup a proxy, then it will show
as 'absent' on every interface. You can setup a proxy by opening System
Preferences, clicking on the Network icon, choosing an interface from the list
on the left, clicking the Advanced button in the lower right corner of the
window, clicking the 'Proxies" tab at the top of the window, clicking the
checkbox next to the "Web Proxy (HTTP)" choice, and entering a proxy URL and
port. NOW do you get why we automate this bullshit?).  Also, your list of
network interfaces may not match mine if you have more or less interfaces than
I do.

TADA! `puppet resource` WORKS! ISN'T THAT AWESOME?! WHY AM I TYPING IN CAPS?!

## Prefetching, flushing, caching, and other hard shit

Okay, so up until now we've implemented one half of the equation - we can query
'is' values and `puppet resource` works. What about using this 'more efficient'
method of getting values for a type on the OTHER end of the spectrum? What if
instead of calling setter methods one-by-one to set values for all resources
of a type in a catalog we had a way to do it all at once? Well, such a way
exists, and it's called the `flush` method...but we're getting slightly ahead
of ourselves. Before we get to flushing, we need to point out that `self.instances`
is ONLY used by `puppet resource` - THAT'S IT (and it's only used by `self.instances`
when you GET values from the system, not when you SET values on the system...and if
you never knew that `puppet resource` could actually SET values on the system, well,
I guess you got another surprise today). If we want `puppet agent` or
`puppet apply` to use the behavior that `self.instances` implements, we need
to create another method: `self.prefetch`

### `self.prefetch`

If you thought `self.instances` didn't have much documentation, wait until
you see `self.prefetch`.  After wading the waters of `self.prefetch`, I'm
PRETTY SURE its implementation might have come to uncle Luke after a long
night in Reed's chem lab where he might have accidently synthesized mescaline.

Let's look at the codebase:

{% codeblock lang:ruby lib/puppet/provider.rb %}
# @comment Document prefetch here as it does not exist anywhere else (called from transaction if implemented)
# @!method self.prefetch(resource_hash)
# @abstract A subclass may implement this - it is not implemented in the Provider class
# This method may be implemented by a provider in order to pre-fetch resource properties.
# If implemented it should set the provider instance of the managed resources to a provider with the
# fetched state (i.e. what is returned from the {instances} method).
# @param resources_hash [Hash<{String => Puppet::Resource}>] map from name to resource of resources to prefetch
# @return [void]
# @api public
{% endcodeblock %}

That's right, documentation for `self.prefetch` in the Puppet codebase is
9 lines of comments in `lib/puppet/provider.rb`, which is awesome. So when
is `self.prefetch` used to provide information to Puppet and when is `self.instances` used? 

|Puppet Subcommand        | Provider Method       | Execution Mode  |
|:-----------------------:|:---------------------:|:---------------:|
| puppet resource         | self.instances        | getting values
| puppet resource         | self.prefetch         | setting values
| puppet agent            | self.prefetch         | getting values
| puppet agent            | self.prefetch         | setting values
| puppet apply            | self.prefetch         | getting values
| puppet apply            | self.prefetch         | setting values  

.  


This doesn't mean that `self.instances` is really only handy for `puppet
resource` - that's definitely not the case.  In fact, frequently you will find
that `self.instances` is used by `self.prefetch` to do some of the heavy
lifting.  Even though `self.prefetch` works VERY SIMILARLY to the way that
`self.instances` works for `puppet resource` (and by that I mean that it's
going to gather a list of instances of a type on the system, and it's also
going to populate `@property_hash` for `puppet apply`, `puppet agent`, and when
when `puppet resource` is setting values), it's not an exact one-for-one match
with `self.instances`.  The `self.prefetch` method for a type is called once
per run when Puppet encounters a resource of that type in the catalog. The
argument to `self.prefetch` is a hash of all managed resources of that type
that are encountered in a compiled catalog for that node (the hash's key will
be the `namevar` of the resource, and the value will be an instance of
`Puppet::Type` - in this case, `Puppet::Type::Mac_web_proxy`).  Your task is to
implement a `self.prefetch` method that gets an array of instances of the
provider that are discovered on the system, iterates through the hash passed to
`self.prefetch` (containing all the resources of the type that were discovered
in the catalog), and passes the correct instance of the provider that was
discovered on the system to the `provider=` method of the correct instance of
the type that was discovered in the catalog.

**What the actual fuck?!**

Okay, let's break that apart to try and discover exactly what's going on here.
Assume that I've setup a proxy for the 'FireWire' interface on my laptop, and
I want to try and manage that resource with `puppet apply` (i.e. something that
uses `self.prefetch`). The resource in the manifest used to manage the proxy
will look something like this:


{% codeblock lang:puppet %}
mac_web_proxy { 'firewire':
  ensure       => 'present',
  proxy_port   => '8080',
  proxy_server => 'proxy.server.org',
}
{% endcodeblock %}

When `self.prefetch` is called by Puppet, it's going to be passed a hash looking
something like this:

{% codeblock lang:ruby %}
{ "firewire" => Mac_web_proxy[firewire] }
{% endcodeblock %}

Because only one resource is encountered in the catalog, only one key/value
pair shows up in the hash that's passed as the argument to `self.prefetch`.

The job of `self.prefetch` is to find the current state of `Mac_web_proxy['firewire']`
on the system, create a new instance of the `mac_web_proxy` provider that contains
the 'is' values for the `Mac_web_proxy['firewire']` resource, and assign this provider instance as the value
of the `provider=` method to the instance of the mac_web_proxy TYPE that is the
VALUE of the 'firewire' key of the hash that's passed to `self.prefetch`.

**No, really, that's what it's supposed to do. I'm not even sure what's real anymore**

You'll remember that `self.instances` gives us an array of resources that were
discovered on the system, so we have THAT part of the implementation written. We also have the
hash of resources that were encountered in the catalog - so we have THAT
part done too. Our only job is to connect the dots (la la la la), programmatically
speaking.  This should just about do it:

{% codeblock lang:ruby %}
def self.prefetch(resources)
  instances.each do |prov|
    if resource = resources[prov.name]
      resource.provider = prov
    end
  end
end
{% endcodeblock %}

I want to make a confession right now - I've only ever copied and pasted this
code into every provider I've ever written that needed `self.prefetch` implemented.
It wasn't until someone actually asked me what it DID that I had to walk the path
of figuring out EXACTLY what it did. Based on the last couple of paragraphs - can
you blame me?

This code iterates through the array of resources returned by `self.instances`,
tries to assign a variable `resource` based on referencing a key in the
`resources` hash using the name of the resource (remember, `resources` is
a hash containing all resources in the catalog), and, if this assignment works
(i.e. it isn't `nil`, which is what happens when you reference a key in a Ruby
hash that doesn't exist), then we're calling the `provider=` method on the
instance of the type that was referenced in the `resources` hash, and passing
it the resource that was discovered on the system by `self.instances`.

Wow.

Why **DID** we do all of that?  We did it all for the `@provider_hash`. Doing
this will populate `@provider_hash` in all methods of the instance of the
provider (i.e. `exists?`, `create`, `destroy`, etc..) just like `self.instances`
did for `puppet resource`.

## Flush it; Ship it

As I alluded to above, the opposite side of the coin to prefetching (which
is a way to query the state for all resources at once) is flushing (or
specifically the `flush` method). The `flush` method is called once per
resource whenever the 'is' and 'should' values for a property differ (and
synchronization needs to occur). The `flush` method **does not take the place
of property setter methods**, but, rather, is used in conjunction with them
to determine how to synchronize resource property values. In this vein, it's
a single trigger that can be used to set all property values for an individual
resource simultaneously.

There are a couple of strategies for implementing `flush`, but one of the more
popular ones in use is to create an instance variable that will hold values to
be synchronized, and then determine inside `flush` how best to make
as-few-as-possible calls to the system to synchronize all the property values
for an individual resource.

Our resource type is unique because the `networksetup` binary that we'll be
using to synchronize values allows us to set most every property value with
a single command. Because of this, we really only need that instance variable
for one property - the ensure value.  But let's start with the initialization
of that instance variable for the `flush` method:

{% codeblock lang:ruby %}
def initialize(value={})
  super(value)
  @property_flush = {}
end
{% endcodeblock %}

The `initialize` method is magic to Ruby - it's invoked when you instantiate
a new object. In our case, we want to create a new instance variable -
`@property_flush` - that will be available to all methods of the instance of
the provider. This instance variable will be a hash and will contain all the
'should' values that will need to be synchronized for a resource. The `super`
method in Ruby sends a message to the parent of the current object, asking it
to invoke a method of the same name (e.g. `intialize`).  Basically, the
`initialize` method is doing the exact same thing as it has always done with
one exception - making the instance variable available to all methods of the
instance of the provider.

### The only 'setter' method you need

This provider is going to be unique not only because the `networksetup`
binary will set values for ALL properties, but because to change/set ANY
property values you have to change/set ALL the property values at the same
time. Typically, you'll see providers that will need to pass arguments to
a binary in order to set individual values.  For example, if you had a
binary `fooset` that took arguments of `--bar` and `--baz` to set values
respectively for `bar` and `baz` properties of a resource, you might see the following
setter and flush methods for `bar` and `baz`:

{% codeblock lang:ruby %}
def bar=(value)
  @property_flush[:bar] = value
end

def baz=(value)
  @property_flush[:baz] = value
end

def flush
  array_arguments = []
  if @property_flush
    array_arguments << '--bar' << @property_flush[:bar] if @property_flush[:bar]
    array_arguments << '--baz' << @property_flush[:baz] if @property_flush[:baz]
  end
  if ! array_arguments.empty?
    fooset(array_arguments, resource[:name])
  end
end
{% endcodeblock %}

That's not the case for `networksetup` - in fact, one of the ONLY places in our code
where we're going to throw a value inside `@property_flush` is going to be in
the `destroy` method. If our intention is to ensure a proxy absent (or, in
this case, disable the proxy for a network interface), then we can short-circuit
the method we're going to create to set proxy values by simply checking for
a value in `@property_flush[:ensure]`. Here's what the `destroy` method looks like:

{% codeblock lang:ruby %}
def destroy
  @property_flush[:ensure] = :absent
end
{% endcodeblock %}

Next, we need a method that will set values for our proxy. This method will
handle all interaction to `networksetup`.  So, how do you set proxy values
with `networksetup`?

    networksetup -setwebproxy <networkservice> <domain> <port number> <authenticated> <username> <password>

The three properties to our `mac_web_proxy` type are `proxy_port`, `proxy_server`,
and `proxy_authenticated` which map to the '\<port number\>', '\<domain\>',
and '\<authenticated\>' values in this command. To change any of these values
means we have to pass ALL of these values (again, which is why our `flush`
implementation may be unique from other `flush` implementations). Here's what
the `set_proxy` method looks like:

{% codeblock lang:ruby %}
def set_proxy
  if @property_flush[:ensure] == :absent
      networksetup(['-setwebproxystate', resource[:name], 'off'])
      return
  end

  if (resource[:proxy_server].nil? or resource[:proxy_port].nil?)
    raise Puppet::Error, "Proxy types other than 'auto' require both a proxy_server and proxy_port setting"
  end
  if resource[:proxy_authenticated] != :true
    networksetup(
      [
        '-setwebproxy',
        resource[:name],
        resource[:proxy_server],
        resource[:proxy_port]
      ]
    )
  else
    networksetup(
      [
        '-setwebproxy',
        resource[:name],
        resource[:proxy_server],
        resource[:proxy_port],
        'on',
        resource[:authenticated_username],
        resource[:authenticated_password]
      ]
    )
  end
  networksetup(['-setwebproxystate', resource[:name], 'on'])
end
{% endcodeblock %}

This helper method does all the validation checks for required properties,
executes the correct command, and enables the proxy. Now, let's implement
`flush`:

{% codeblock lang:ruby %}
def flush
  set_proxy

  # Collect the resources again once they've been changed (that way `puppet
  # resource` will show the correct values after changes have been made).
  @property_hash = self.class.get_proxy_properties(resource[:name])
end
{% endcodeblock %}

The last line re-populates `@property_hash` with the current resource values,
and is necessary for `puppet resource` to return correct values after it
makes a change to a resource during a run.

### The final method

We've implemented logic to query the state of all resources, to prefetch those
states, to make changes to all properties at once, and to destroy a resource
if it exists, but we've yet to implement logic to CREATE a resource if it
doesn't exist and it should.  Well, this is a bit of a lie - the logic is in
the code, but we don't have a `create` method, so Puppet's going to complain:

{% codeblock lang:ruby %}
def create
  @property_flush[:ensure] = :present
end
{% endcodeblock %}

Technically, this method doesn't have to do a DAMN thing. Why? Remember how the
`flush` method is triggered when a resource's 'is' values differ from its
'should' values? Also, remember how the `flush` method only calls the `set_proxy`
method? And, finally, remember how `set_proxy` only checks if
`@property_flush[:ensure] == :absent` (and if it doesn't, then it goes about
its merry way running `networksetup`)?  Right, well add these things up and
you'll realize that the `create` method is essentially meaningless based on our
implementation (but if you OMIT `create`, then Puppet's going to throw a shit-fit
in the shape of of a `Puppet::Error` exception):

    Error: /Mac_web_proxy[firewire]/ensure: change from absent to present failed: Could not set 'present' on ensure: undefined method `create' for Mac_web_proxy[firewire]:Puppet::Type::Mac_web_proxy

So make Puppet happy and write the goddamn `create` method, okay?

## The complete provider:

Wow, that was a wild ride, huh?  If you've been coding along, you should have 
created a file that looks something like this:

{% codeblock lang:ruby lib/puppet/provider/mac_web_proxy/ruby.rb %}
Puppet::Type.type(:mac_web_proxy).provide(:ruby) do
  commands :networksetup => 'networksetup'

  mk_resource_methods

  def initialize(value={})
    super(value)
    @property_flush = {}
  end

  def self.get_list_of_interfaces
    interfaces = networksetup('-listallnetworkservices').split("\n")
    interfaces.shift
    interfaces.sort
  end

  def self.get_proxy_properties(int)
    interface_properties = {}

    begin
      output = networksetup(['-getwebproxy', int])
    rescue Puppet::ExecutionFailure => e
      Puppet.debug "#get_proxy_properties had an error -> #{e.inspect}"
      return {}
    end

    output_array = output.split("\n")
    output_array.each do |line|
      line_values = line.split(':')
      line_values.last.strip!
      case line_values.first
      when 'Enabled'
        interface_properties[:ensure] = line_values.last == 'No' ? :absent : :present
      when 'Server'
        interface_properties[:proxy_server] = line_values.last.empty? ? nil : line_values.last
      when 'Port'
        interface_properties[:proxy_port] = line_values.last == '0' ? nil : line_values.last
      when 'Authenticated Proxy Enabled'
        interface_properties[:proxy_authenticated] = line_values.last == '0' ? nil : line_values.last
      end
    end

    interface_properties[:provider] = :ruby
    interface_properties[:name]     = int.downcase
    Puppet.debug "Interface properties: #{interface_properties.inspect}"
    interface_properties
  end

  def self.instances
    get_list_of_interfaces.collect do |int|
      proxy_properties = get_proxy_properties(int)
      new(proxy_properties)
    end
  end

  def create
    @property_flush[:ensure] = :present
  end

  def exists?
    @property_hash[:ensure] == :present
  end

  def destroy
    @property_flush[:ensure] = :absent
  end

  def self.prefetch(resources)
    instances.each do |prov|
      if resource = resources[prov.name]
        resource.provider = prov
      end
    end
  end

  def set_proxy
    if @property_flush[:ensure] == :absent
        networksetup(['-setwebproxystate', resource[:name], 'off'])
        return
    end

    if (resource[:proxy_server].nil? or resource[:proxy_port].nil?)
      raise Puppet::Error, "Both the proxy_server and proxy_port parameters require a value."
    end
    if resource[:proxy_authenticated] != :true
      networksetup(
        [
          '-setwebproxy',
          resource[:name],
          resource[:proxy_server],
          resource[:proxy_port]
        ]
      )
    else
      networksetup(
        [
          '-setwebproxy',
          resource[:name],
          resource[:proxy_server],
          resource[:proxy_port],
          'on',
          resource[:authenticated_username],
          resource[:authenticated_password]
        ]
      )
    end
    networksetup(['-setwebproxystate', resource[:name], 'on'])
  end

  def flush
    set_proxy

    # Collect the resources again once they've been changed (that way `puppet
    # resource` will show the correct values after changes have been made).
    @property_hash = self.class.get_proxy_properties(resource[:name])
  end
end
{% endcodeblock %}

Undoubtedly there are better ways to write this Ruby code, no? Also, I'm SURE
I have some errors/bugs in that code. It's those things that keep me in a job...

## Final Thoughts

So, I write these posts not to belittle or mock anyone who works on Puppet
or wrote any of its implementation (except the amazing/terrifying bastard who
came up with `self.prefetch`). Anybody who contributes to open source and who
builds a tool to save some time for a bunch of sysadmins is fucking awesome
in my book.

No, I write these posts so that you can understand the **'WHY'** piece of the
puzzle. If you fuck up the **'HOW'** of the code, you can spend some time in
Google and IRB to figure it out, but if you don't understand the **'WHY'**
then you're probably not going to even bother.

Also, selfishly, I move from project to project so quickly that it's REALLY
easy to forget both why AND how I did what I did. Posts like these give me
someplace to point people when they ask me "What's `self.prefetch`?" that
ISN'T just the source code or a liquor store.

This isn't the last post in the series, by the way. I haven't even TOUCHED
on writing unit tests for this code, so that's going to be a WHOLE other
piece altogether. Also, while this provider manages a **WEB** proxy for
a network interface, understand that there are MANY MORE kinds of proxies
for OS X network interfaces (including socks and gopher!). A future post
will show you how to refactor the above into a parent provider that can
be inherited to allow for code re-use among all the proxy providers that I
need to create.

As always, you're more than welcome to comment, ask questions, or simply bitch
at me both on this blog as well as [on Twitter: @glarizza][twitter]. Hopefully
this post helped you out and you learned a little bit more about how Puppet
providers do their dirty work...

Namaste, bitches.


[firstpost]: http://garylarizza.com/blog/2013/11/25/fun-with-providers/
[secondpost]: http://garylarizza.com/blog/2013/11/26/fun-with-providers-part-2/
[puppetbook]: http://www.amazon.com/Puppet-Types-Providers-Dan-Bode/dp/1449339328/ref=sr_1_1?ie=UTF8&qid=1387136838&sr=8-1&keywords=types+and+providers+puppet+book
[resourcesresource]: http://docs.puppetlabs.com/references/latest/type.html#resources
[twitter]: http://www.twitter.com/glarizza
