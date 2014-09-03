---
layout: post
title: "Who abstracted my Ruby?"
date: 2013-11-26 16:01
comments: true
categories:  ['os x', 'puppet', 'types', 'providers', 'proxies']
---

Previously, on Lost, I said a lot of words about Puppet Types;
[you should totally check it out.](http://garylarizza.com/blog/2013/11/25/fun-with-providers/)
In this second installment, you're going to find out how to actually throw
pure Ruby at Puppet in a way that makes you feel accomplished. And useful. And elitist.
Well, possibly just elitist. Either way, read on - there's much thought-leadership to be done...

In the last post, we learned that Types will essentially dictate the attributes
that you'll be passing in your resource declaration using the DSL. In the
simplest and crudest explanation I could muster, types model how your
declaration will look in the manifest. Providers are where the actual
IMPLEMENTATION happens. If you've ever wondered how this:

{% codeblock lang:puppet %}
package { 'httpd':
  ensure => installed,
}
{% endcodeblock %}

eventually gets turned into this:

{% codeblock lang:bash %}
yum install -e 0 -d 0 -y httpd
{% endcodeblock %}

your answer would be "It's in the provider file".

## Dirty black magic

I've seen people do the craziest shit imaginable in the Puppet DSL simply because
they're:

* Unsure how types and providers work
* Afraid of Ruby
* Confused by error messages
* Afraid to ask for help

Sometimes you have a problem that can only be solved by interacting with data
that's returned by a binary (using some binary to get a value, and then using
that binary to set a value, and so on...). I see people writing defined resource
types with a SHIT TON of `exec` statements and conditional logic to model this
data when a type and provider would not only BETTER model the problem but would
also be shareable and re-useable by other folk. The issue is that while the DSL
is REALLY easy to get started with, types and providers still feel like dirty
black magic.

The reason is because they're dirty black magic.

Hopefully, I can help get you over the hump and onto a working implementation. Let's
take a problem I had last week:

## Do this if that, and then be done

I was working with a group who wanted to set a list of domains that would
bypass their web proxy for a specific network interface on an OS X workstation.
It sounds so simple, because it was. Due to the amount of time I had on-site,
I wrote a class with some nasty `exec` statements, a couple of facts, and some
conditional logic because that's what you do when you're in a hurry...but it
doesn't make it right. When I left, I hacked up a type and provider, and it's
a GREAT example because you probably have a similar problem.  Let's look at the
information we have:

The list of network interfaces:

{% codeblock lang:bash %}
└▷ networksetup -listallnetworkservices
An asterisk (*) denotes that a network service is disabled.
Bluetooth DUN
Display Ethernet
Ethernet
FireWire
Wi-Fi
iPhone USB
Bluetooth PAN
{% endcodeblock %}

Getting the list of bypass domains for an interface:

{% codeblock lang:bash %}
└▷ networksetup -getproxybypassdomains Ethernet
www.garylarizza.com
*.corp.net
10.13.1.3/24
{% endcodeblock %}

The message displayed when no domains are set for an interface:

{% codeblock lang:bash %}
└▷ networksetup -getproxybypassdomains FireWire
There aren't any bypass domains set on FireWire.
{% endcodeblock %}

Setting the list of bypass domains for an interface:

{% codeblock lang:bash %}
└▷ networksetup -setproxybypassdomains Ethernet '*.corp.net' '10.13.1.3/24' 'www.garylarizza.com'
{% endcodeblock %}

Perfect - all of that is done with a single binary, and it's pretty straightforward. Let's look at the type I ended up creating for this problem:

{% codeblock lib/puppet/type/mac_proxy_bypassdomains.rb lang:ruby %}
Puppet::Type.newtype(:mac_proxy_bypassdomains) do
  desc "Puppet type that models bypass domains for a network interface on OS X"

  ensurable

  newparam(:name, :namevar => true) do
    desc "Interface name - currently must be 'friendly' name (e.g. Ethernet)"
  end

  newproperty(:domains, :array_matching => :all) do
    desc "Domains which should bypass the proxy"
    def insync?(is)
      is.sort == should.sort
    end
  end
end
{% endcodeblock %}

The type uses a namevar parameter called 'name', which is the name of the network interface.
This means that we can set one list of bypass domains for every network interface.
There's a single property, 'domains' that accepts an array of domains that should bypass
the proxy for the network interface. I've overridden the `insync?` method for the domains
property to sort the array values on both ends - this means that the ORDER of the domains
doesn't matter, I only care that the domains specified exist on the system. Finally, the
type is ensurable (which means that we can create a list of domains and remove/destroy
the list of domains for a network interface).

## Setup the provider

Okay, so we've defined the problem, seen how to interact with the system to get us the
data that we need, setup a type to model the data, and now the last thing left to do
is to wire up the provider to make the binary calls we need and return the data we
want.

### Typos are not your friend.

The first thing you will encounter is "Puppet's predictable naming pattern"
that is used by the Puppet autoloader. Typos are not fun, and omitting a single
letter in either the filename or the provider name will render your provider
(emotionally) unavailable to Puppet. Our type is called 'mac_proxy_bypassdomains',
as types are generally named along the lines of 'what does this data model?' The
provider name is generally the name of the underlying technology that's doing
the modeling. For the package type, the providers are named after the package
management systems (e.g. yum, apt, pacman, zypper, pip), for the file type, the
providers are loosely named for the operatingsystem kernel type on which files
are to be created (e.g. windows, posix). In our example, I simply chose to name
the provider 'ruby' because, as a Puppet Labs employee, **I TOO** suck at naming
things.

Here's a tree of my module to understand how the type and provider files are
to be laid out:

{% codeblock Module tree lang:bash %}
├── Modulefile
├── README.markdown
└── lib
    └── puppet
        ├── provider
        │   ├── mac_proxy_bypassdomains
        │   │   └── ruby.rb
        └── type
            └── mac_proxy_bypassdomains.rb
{% endcodeblock %}

As you can see from above, the name of both the type and provider must **EXACTLY**
match the filename of their corresponding files. Also, the provider file lives
in a directory named after the type. There are MANY things that can be typoed here
(filenames, foldernames, type/provider names in their files), so be absolutely sure
that you've named your files correctly.

The reason for all this naming bullshit is because of the way Puppet syncs down
plugin files (coincidentally, with a [process known as Pluginsync)](http://docs.puppetlabs.com/guides/plugins_in_modules.html).
Everything in the `lib` directory in a Puppet module is going to get synced down
to your nodes inside [the `vardir` directory](http://docs.puppetlabs.com/references/latest/configuration.html#vardir) on
the node itself. The `vardir` is a known library path to Puppet, and all files
in the `vardir` are treated as if they had lived in Puppet's source code (in
the same relative paths).  Because the Puppet source code has all type files in
the `lib/puppet/type` directory, all **CUSTOM** types must go in the module's
`lib/puppet/type` directory for confirmity.  This is repeated for **EVERY**
custom Puppet/Facter plugin (including custom facts, custom functions, and
etc...).

### More scaffolding

Let's layout the shell of our provider, first, to ensure that we haven't typoed
anything. Here's the provider declaration:

{% codeblock lib/puppet/type/mac_proxy_bypassdomains/ruby.rb lang:ruby %}
Puppet::Type.type(:mac_proxy_bypassdomains).provide(:ruby) do
  # Provider work goes here
end
{% endcodeblock %}

Note that the name of the type and the name of the provider are symbolized (i.e.
they're prepended with a colon). Like I mentioned above, they must be spelled
EXACT or Puppet will complain very loudly. You may see variants on that
declaration line because there are multiple ways in Ruby to extend a class
object. The method I've listed above is the 'generally accepted best-practice', which
is to say it's the way we're doing it this month.

Congrats! You have THE SHELL of a provider that has yet to do a single goddamn thing!
Technically, you're further than about 90% of other Puppet users at this point! Let's
go the additional 20% (since we're basing this on a mangement metric of 110%) by
wiring up the methods and making the damn thing work!

### Are you (en)sure about this?

We've explained before that a type is 'ensurable' when you can check for its existance
on a system, create it when it doesn't exist (and it SHOULD exist), and destroy it when
it does exist (and it SHOULDN'T exist). The bare minimum amount of methods necessary
to make a type ensurable is three, and they're called `exists?`, `create`, and `destroy`.

### Method: `exists?`

The `exists?` method is a predicate method - that means it should either return
the boolean `true` or `false` value based on whether the bypass domain list
exists. Puppet will always call the `exists?` provider method to determine if
that 'thing' (in this case, 'thing' means 'a list of domains to bypass for a
specific network interface') exists before calling any other methods. How do we
know if this thing exists? Like I showed before, you need to run the
`networksetup -getproxybypassdomains` command and pass the interface name.  If
it returns 'There aren't any bypass domains set on (interface name)', then the
list doesn't exist. Let's do some binary execution...

#### Calling binaries from Puppet

Puppet provides some helper syntax around basic actions that most providers
perform. MOST providers are going to need to call out to an external binary
(e.g. yum, apt, etc...) at some point, and so Puppet allows you to create
your own method JUST for a system binary. The `commands` method abstracts
all the dirtyness of making a method for each system binary you want to call.
The way you use the `commands` method is like so:

{% codeblock lang:ruby %}
commands :networksetup => 'networksetup'
{% endcodeblock %}

The `commands` method accepts a hash whose key must be a symbolized name. The
CONVENTION is to use a symbolized name that matches the binary name, but it's
not REQUIRED to do so. The value for that symbolized key MUST be the binary
name. Note that I've not passed a full path to the binary. Why? Well, Puppet
will automatically do a path lookup for that binary and store its full path
for use when the binary is invoked. We don't REQUIRE you to pass the full
path because sometimes the same binary exists in different locations for
different operatingsystems. Instead of creating a provider for each OS you
manage with Puppet, we abstract away the path stuff. You **CAN** still
pass a full path as a value, but if you elect to do that an the binary doesn't
exist at that path, Puppet will disqualify the provider and you'll be quite
upset.

In the event that Puppet **CANNOT** find this binary, it will disqualify the
entire provider, and you'll get a message saying as much in the debug output
of your Puppet run. Because of that, the `commands` method is a good way to
confine your provider to a specific system or class of system.

When the `commands` method is successfully invoked, you will get a new provider
method named after the **SYMBOLIZED** key, and not necessarily the binary
name (unless you made them the same). After the above command is evaluated,
Puppet will now have a `networksetup()` method in our provider. The argument
to the `networksetup` method should be an array of arguments that are passed
to the binary. It's c-style, so each element is going to be individually
quoted. You can run into issues here if you pass values containing quotes
as part of your argument array. Read that again - quoting your values is
totally acceptable (e.g. ['foo', 'bar']), but passing a value that contains
quotes can potentially cause problems (e.g. ["'foo'", "'bar'"]).

You're probably thinking "Why the hell would I go through this trouble when
I can use the `%x{}` syntax in ruby to execute a shell command?!" And to that
I would say "Quit yelling at me" and also "Because: testing." When you write
spec tests for your provider (which will be covered in a later blog post, since
it's its OWN path of WTF), you're going to need to mock out calls to the system
during your tests (i.e. sometimes you may be running the tests on a system that
doesn't have the binary you're meant to be calling in your provider. You don't
want the tests to fail due to the absence of a binary file). The `%x{}`
construct in Ruby is hard to mock out, but a method of our provider is a relatively
easy thing to mock out. Also - see the path problem above. We don't STOP you
from doing `%x{}` in your code (it will still totally work), but we give you
a couple of good reasons to NOT do it.

#### Objects are a provider's best friend

Within your provider, you're going to be doing lots of system calls and data
manipulation. Often we're asked whether you do that ugliness inside the main
methods (i.e. inside the `exists?` method directly), or if you create a
helper method for some of this data manipulation. The answer I usually give
is that you should probably create a helper method if:

* The code is going to be called more than once
* The code does something that would be tricky to test (like reading from a file)
* Complexity would be reduced by creating a helper method

The act of getting a list of domains for a specific interface is definitely
going to be utilized in more than one place in our provider (we'll use it
in the `exists?` method as well as in a 'getter' method for the `domains`
property). Also, you could argue that it might be tricky to test since it's
going to be a binary call that's going to return some data. Because of this,
let's create a helper method that returns a list of domains for a specific
interface:

{% codeblock lang:ruby %}
def get_proxy_bypass_domains(int)
  begin
    output = networksetup(['-getproxybypassdomains', int])
  rescue Puppet::ExecutionFailure => e
    Puppet.debug("#get_proxy_bypass_domains had an error -> #{e.inspect}")
    return nil
  end
  domains = output.split("\n").sort
  return nil if domains.first =~ /There aren\'t any bypass domains set/
  domains
end
{% endcodeblock %}

Ruby convention is to use underscores (i.e. versus camelCase or hyphens) in
method names. You want to give your methods very descriptive names based on
what it is that they DO. In this case, `get_proxy_bypass_domains` seems
adequately descriptive. Also, you should err on the side of readability when
you're writing code. You can get pretty creative with Ruby metaprogramming,
but that can quickly become hard to follow (and then you're just a dick).
Finally, error-handling is a good thing. If you're going to do any error-handling,
though, be very specific about the errors you catch/rescue. When you have a rescue
block, make sure you catch a specific exception class (in the case above, we're
catching a Puppet::ExecutionFailure - which means the binary is returning a
non-zero exit code). 

The code above will return an array containing all the domains, or it will
return `nil` if domains aren't found or the `networksetup` binary had an issue.

Using the helper method above, here's what the final `exists?` method looks like:

{% codeblock lang:ruby %}
def exists?
  get_proxy_bypass_domains(resource[:name]) != nil
end
{% endcodeblock %}

All provider methods have the ability to access the 'should' values for the resource
(and by that I mean the values that are set in the Puppet maniest on the Puppet master
server, or locally if you're using `puppet apply`). Those values reside in the `resource`
method that responds with a hash. In the code above, `resource[:name]` will return the
network interface name (e.g. Ethernet, FireWire, etc...) that was specified in the
Puppet manifest. The exists method will return true of a list of domains exists for
an interface, or it will return false if a list of domains does not exist (i.e.
`get_proxy_bypass_domains` returns `nil`).

### Method: `create`

The `create` method is called when `exists?` returns false and a resource has an
`ensure` value set to `present`. Because of this, you don't need to call the `exists?`
method explicitly in `create` - it's already been evaluated. Remember from above that
the `-setproxybypassdomains` argument to the `networksetup` binary will set a domain
list, so the `create` method is going to be very short-and-sweet:

{% codeblock lang:ruby %}
def create
  networksetup(['-setproxybypassdomains', resource[:name], resource[:domains]])
end
{% endcodeblock %}

In the end, the `create` method will call the `networksetup` binary with the `-setproxybypassdomains`
argument, pass the interface name (from `resource[:name]`) and pass an array of domain values (which
comes from `resource[:domains]`). That's it; it's done!

### Method: `destroy`

The `destroy` method is easier than the `create` method:

{% codeblock lang:ruby %}
def destroy
  networksetup(['-setproxybypassdomains', nil])
end
{% endcodeblock %}

Here, we're calling `networksetup` with the `-setproxybypassdomains` argument
and passing nothing else. This will initialize the list and set it to be empty.

## Synchronizing properties

### Getter method: `domains`

At this point our type is ensurable, which means we can create and destroy resources.
What we CAN'T do, however, is change the value of any properties that are out-of-sync. A
property is out-of-sync when the value discovered by Puppet on the node differs from the
value in the catalog (i.e. set by the Puppet manifest using the DSL on the Puppet master).
Just like `exists?` is called to determine if a resource exists, Puppet needs a way to
get the current value for a property on a node. The method that gets this value is
called the 'getter method' for a property, and its name must match the name of the
property. Because we have a property called `domains`, the provider must have a `domains`
method that returns a value (in this case, an array of domains to be bypassed by the
proxy). We've already written a helper method that does this work for us, so the
`domains` getter method is pretty easy:

{% codeblock lang:ruby %}
def domains
  get_proxy_bypass_domains(resource[:name])
end
{% endcodeblock %}

Tada! Just call the helper method and pass the interface name. Boom - instant array
of values. The getter method will return the 'is' value, because that's what the value
**IS** (currently on the node). Get it? Anyone? The **IS** value is the other side of
the coin to the 'should' value (that comes from the Puppet manifest) because that's
what the value **SHOULD** be set on the node.

### Setter method: `domains=`

If the getter method (e.g. `domains`) returns a value that doesn't match the value
in the catalog, then Puppet changes the value on the node and sets it to the value
in the catalog. It does this by calling the 'setter' method for the property, which
is the name of the property and the equals ( = ) sign. In this case, the setter
method for the `domains` property must be called `domains=`.  It looks like this:

{% codeblock lang:ruby %}
def domains=(value)
  networksetup(['-setproxybypassdomains', resource[:name], value])
end
{% endcodeblock %}

Setter methods are always passed a single argument - the 'should' value of the property.
In our example, we're calling the `networksetup` binary with the `-setproxybypassdomains`
argument, passing the name of the interface, and then passing the 'should' value - or
the array of domains. It's easy, it's one line, and I love it when a plan comes together

## Putting the whole damn thing together

I've broken down the provider line by line, but here's the entire file:

{% codeblock lib/puppet/provider/mac_proxy_bypassdomains/ruby.rb lang:ruby %}
Puppet::Type.type(:mac_proxy_bypassdomains).provide(:ruby) do
  commands :networksetup => 'networksetup'

  def get_proxy_bypass_domains(int)
    begin
      output = networksetup(['-getproxybypassdomains', int])
    rescue Puppet::ExecutionFailure => e
      Puppet.debug("#get_proxy_bypass_domains had an error -> #{e.inspect}")
      return nil
    end
    domains = output.split("\n").sort
    return nil if domains.first =~ /There aren\'t any bypass domains set/
    domains
  end

  def exists?
    get_proxy_bypass_domains(resource[:name]) != nil
  end

  def destroy
    networksetup(['-setproxybypassdomains', nil])
  end

  def create
    networksetup(['-setproxybypassdomains', resource[:name], resource[:domains]])
  end

  def domains
    get_proxy_bypass_domains(resource[:name])
  end

  def domains=(value)
    networksetup(['-setproxybypassdomains', resource[:name], value])
  end
end
{% endcodeblock %}

## Testing the type/provider

And that's it, we're done!  The last thing to do is to test it out.  You can
test out your provider in one of two ways: the first is to add the module to
the modulepath of your Puppet master and include it that way, or test it
locally by setting the `$RUBYLIB` environmental variable to point to the `lib`
directory of your module (which is the more preferred method since it won't
serve it out to all of your nodes without it being tested). Because this module
is on my system at `/users/glarizza/src/puppet-mac_proxy`, here's how my
`$RUBYLIB` is set:

{% codeblock lang:bash %}
export RUBYLIB=/users/glarizza/src/puppet-mac_proxy/lib
{% endcodeblock %}

Next, we need to create a resource declaration to try and set a couple of
bypass domains. I'll create a `tests` directory and simple test file in
`tests/mac_proxy_bypassdomains.pp`:

{% codeblock tests/mac_proxy_bypassdomains.pp lang:puppet %}
mac_proxy_bypassdomains { 'Ethernet':
  ensure  => 'present',
  domains => ['www.garylarizza.com','*.puppetlabs.com','10.13.1.3/24'],
}
{% endcodeblock %}

Finally, let's run Puppet and test it out:

{% codeblock %}
└▷ puppet apply ~/src/puppet-mac_proxy/tests/mac_proxy_bypassdomains.pp
Notice: Compiled catalog for satori.local in environment production in 0.06 seconds
Notice: /Stage[main]//Mac_proxy_bypassdomains[Ethernet]/domains: domains changed [] to 'www.garylarizza.com *.puppetlabs.com 10.13.1.3/24'
Notice: Finished catalog run in 3.47 seconds
{% endcodeblock %}

**NOTE:** If you run this as a local user, you will be prompted by OS X to
enter an administrative password for a change.  Since Puppet will ultimately be
run as root on OS X when we're NOT testing out code, this shouldn't be required
during a normal Puppet run. To test this out (i.e. that you don't always have to
enter an admin password in a pop-up window), you'll need to `sudo -s` to
change to root, set the `$RUBYLIB` as the root user, and then run Puppet again.

And that's it - looks like our code worked! To check and make sure it will notice a
change, open System Preferences, then the Network pane, click on the Ethernet
interface, then the Advanced button, then the Proxies tab, and finally note the
'Bypass proxy settings...' text box at the bottom of the screen (now do you see
why we automate this shit?!). Make a change to the entries in there and run
Puppet again - it should correct it for you

## Wait...so that was it?  Really?  We're done?

Yeah, that was a whole type and provider. Granted, it has only one property and it's
not too complicated, but that's the point. We've still got some latent bugs (the
network interface passed must be capitalized exactly like OS X expects it, we could
do some better error handling, etc...), and the type doesn't work with `puppet resource`
(yet), but we'll handle all of these things in the next blog post (or two...or three).

Until then, take this time to crack open a type and a provider for something that's
been pissing you off and FIX it!  Better yet, push it up to Github, tweet about it,
and post it up [on The Forge](http://forge.puppetlabs.com) so the rest of the
community can use it!

Like always, feel free to comment, tweet me (@glarizza), email me (gary **AT** puppetlabs **DOT** com),
or use the social media platform of choice to get a hold of me (Snapchats may or may not
get a response. Maybe.)  Cheers!
