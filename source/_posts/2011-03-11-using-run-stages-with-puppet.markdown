---
layout: post
title: "Using Run Stages with Puppet"
date: 2011-03-11 11:34
comments: true
categories: ['puppet', 'runstages']
---


As of version 2.6, Puppet introduced a feature called "Run Stages" that will allow you to better control the order that Puppet configures your system.  The problem with Run Stages (as of right now) is that there's not that much good Documentation out there.  Hopefully this document helps someone else out there who wants to setup Run Stages in their own environment.

##A NOTE ON SYNTAX!!!

One of THE MOST DIFFICULT concepts to understand for puppet newbies is that these two things are identical:

```puppet
    include foo
    class { 'foo': }
```

Both of these lines INVOKE the 'foo' class for use on a particular node.  The problem is that the second method (the parameterized class method) looks nearly IDENTICAL to the method you would use to DECLARE the foo class in foo.pp.  The only difference is that the class name is now INSIDE the curly braces (versus being OUTSIDE the braces when you DECLARE the class in your foo.pp file).  It is VERY important that you distinguish the difference between DECLARING a class and using a parameterized class to INCLUDE the class in your site.pp file.  I will point this out later too... 

##A Linux Example - Repositories

I would bet that the resource with the most "require" and "before" dependencies in the Linux world (Well, in the RHEL world anyways) would be the yumrepo.  I'm sure most of us have many declared yumrepo resources in our manifests that each have their own tangled web of dependencies.  My goal was to create a "repo" class that would have all of my repositories and use a Run Stage to ensure that my repo class was installed prior to any other package installs.  Let's look at some code:

```puppet
class general::repo {
  yumrepo { 'huronrepo':
    descr    => 'Huron Repository',
    enabled  =>  '1',
    gpgcheck =>  '0',
    baseurl  =>  'http://10.13.0.6/huronrepo',
  }
}
```

This is my repo subclass based on my general base class.  I have a single repository, but could easily have another 5 or 10 as need arises (I'm keeping things simple for demo purposes).  Let's look at another class:

```puppet
class general::centos {
  include general::repo
  
  package { 'mcollective':
    name    => "mcollective",
    ensure  => 'present',
    require => Package['stomp'],
  }
}
```

Here's a centos subclass off of the general base class.  We're including our repo class and declaring a single package that requires our "huronrepo" yumrepo.  If we only needed to install a single package, we could use the "require" parameter; but if you assume that we will eventually need to install MULTIPLE packages, then this logic doesn't scale. Just to keep things consistent, here's my general base class:

```puppet
class general {
  case $::operatingsystem {
    'CentOS': { include general::centos }
    'Darwin': { include general::osx }
  }
}
```

You can see that we include the general::centos class as long as Puppet is running on a CentOS box.  There's one final piece to this puzzle - it's the actual declaration and assignment of the Run Stages.  I'm doing this in my site.pp file.  Here's a snippit of what's in my site.pp file:

```puppet
# /etc/puppet/manifests/site.pp

Exec { path =&gt; "/usr/bin:/usr/sbin:/bin:/sbin" }

# Run Stages
stage { 'pre':
  before => Stage["main"],
}

# Node Declaration
node 'server.whomever.com' {
  class { 'general::repos': 
    stage => 'pre'
  }
}
```

Here's where we've declared a "pre" stage that needs to run before the "main" stage.  We don't have to declare the "main" stage because Puppet uses that stage by default.  Next, we're including the general::repos class inside our 'server.whomever.com' node declaration and assigning it to the "pre" stage (which means that everything in that class will get configured prior to our "main" stage).  Remember that you can declare as many stages as you need and chain them all to setup the order that you want, but that can become very unmanageable very quickly.  I find it ideal to setup a 'pre' stage for repo setup, and then setting up dependencies within class files.

##The World is Your Stage

This might not be the "best" way to use Run Stages, but it works for me.  Hopefully this little writeup cements the concept for you too.
