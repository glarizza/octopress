---
layout: post
title: "Building a Functional Puppet Workflow Part 1: Module Structure"
date: 2014-02-17 14:01:37 -0800
comments: true
categories: ['puppet', 'workflow']
---

Working as a professional services engineer for [Puppet Labs][puppetlabs],
my life consists almost entirely of either correcting some of the worst code
atrocities you've seen in your life, or helping people get started with Puppet
so that they don't need to call us again due to: A.) Said code atrocities or
B.) Refactor the work we JUST helped them start.  It wasn't ALWAYS like this -
I can remember some of my earliest gigs, and I almost feel like I should go
revisit them if only to correct some of the previous 'best practices' that
didn't quite pan out.

This would be exactly why I'm wary of 'Best Practices' - because one person's
'Best Practice' is another person's 'What the fuck did you just do?!'

Having said that, I'm finding myself repeating a story over and over again when
I train/consult, and that's the story of 'The Usable Puppet Workflow.'  Everybody
wants to know 'The Right Way™', and I feel like we finally have a way that survives
a reasonable test of time. I've been promoting this workflow for over a year (which
is a HELL of a long time in Startup time), and I've yet to really see an edge case
it couldn't handle.

(If you're already savvy: yes, this is the Roles and Profiles talk)

I'll be breaking this workflow down into separate blog posts for every component,
and, as always, your comments are welcome...

## It all starts with the component module

The first piece of a functional Puppet deployment starts with what we call
'component modules'.  Component modules are the lowest level in your
deployment, and are modules that configure specific pieces of technology (like
apache, ntp, mysql, and etc...).  Component modules are well-encapsulated, have
a reasonable API, and focus on doing small, specific things really well (i.e.
the \*nix way).

I don't want to write thousands of words on building component modules because
I feel like others have done this better than I. As examples, check out 
[RI's Post on a simple module structure][rimodule], 
[Puppet Labs' very own docs on the subject][docsmodule], 
[and even Alessandro's Puppetconf 2012 session.][almodule] Instead, I'd like to
provide some pointers on what I feel makes a good component module, and some
'gotchas' we've noticed.

### Parameters are your API

In the current world of Puppet, you MUST define the parameters your module will
accept in the Puppet DSL. Also, every parameter MUST ultimately have a value
when Puppet compiles the catalog (whether by explicitly passing this parameter
value when declaring the class, or by assuming a default value). Yes, it's
funny that, when writing a Puppet class, if you typo a VARIABLE Puppet will
not alert you to this (in a NON `use strict`-ian sort of approach) and will
happily accept a variable in an undefined state, but the second you don't pass
a value to your class parameter you're in for a rude compilation error.
This is the way of Puppet classes at the time of this writing, so you're going
to see Puppet classes with LINES of defined parameters. I expect this to change
in the future (please let this change in the near future), but for now, it's
a necessary evil.

The parameters you expose to your top-level class (i.e. given class names like
`apache` and `apache::install`, I'm talking specifically about `apache`) should
be treated as an API to your module. IDEALLY, they're the ONLY THING that
a user needs to modify when using your module. Also, whenever possible, it
should be the case that a user need ONLY interact with the top-level class when
using your module (of course, defined resource types like `apache::vhost` are
used on an ad-hoc basis, and thus are the exception here).

### Inherit the `::params` class

We're starting to make enemies at this point. It's been a convention for modules
to use a `::params` class to assign values to all variables that are going to
be used for all classes inside the module. The idea is that the `::params` class
is the one-stop-shop to see where a variable is set. Also, to get access to a
variable that's set in a Puppet class, you have to declare the class (i.e. use
the `include()` function or inherit from that class). When you declare a class
that has both variables AND resources, those resources get put into the catalog,
which means that Puppet ENFORCES THE STATE of those resources. What if you only
needed a variable's value and didn't want to enforce the rest of the resources
in that class? There's no good way in Puppet to do that. Finally, when you inherit
from a class in Puppet that has assigned variable values, you ALSO get access
to those variables in the parameter definition section of your class (i.e. the
following section of the class:

    class apache (
      $port = $apache::params::port,
      $user = $apache::params::user,
    ) inherits apache::params {

See how I set the default value of `$apache::port` to `$apache::params::port`?
I could only access the value of the variable `$apache::params::port` in that
section by inheriting from the `apache::params` class.  I couldn't insert
`include apache::params` below that section and be allowed access to the variable
up in the parameter defaults section (due to the way that Puppet parses classes).

**FOR THIS REASON, THIS IS THE ***ONLY*** RECOMMENDED USAGE OF INHERITANCE IN
PUPPET!**

We do NOT recommend using inheritance anywhere else in Puppet and for any other
reason because there are better ways to achieve what you want to do INSTEAD of
using inheritance.  Inheritance is a holdover from a scarier, more lawless time.

**NOTE: Data in Modules** - There's a 'Data in Modules' pattern out there that
attempts to eliminate the `::params` class.  [I wrote about it in a previous post][hierablog],
and I recommend you read that post for more info (it's near the bottom).

### Do **NOT** do Hiera lookups in your component modules!

This is something that's really only RECENTLY been pushed. When Hiera was
released, we quickly recognized that it would be the answer to quite a few
problems in Puppet. In the rush to adopt Hiera, many people started adding
Hiera calls to their modules, and suddenly you had 'Hiera-compatible' modules
out there. This caused all kinds of compatibility problems, and it was largely
because there wasn't a better module structure and workflow by which to integrate
Hiera. The pattern that I'll be pushing DOES INDEED use Hiera, **BUT** it
confines all Hiera calls to a higher-level wrapper class we call a 'profile'.
The reasons for NOT using Hiera in your module are:

* By doing Hiera calls at a higher level, you have a greater visibility on
exactly what parameters were set by Hiera and which were set explicitly or by
default values.
* By doing Hiera calls elsewhere, your module is backwards-compatible for
those folks who are NOT using Hiera

Remember - your module should just accept a value and use it somewhere. Don't
get **TOO** smart with your component module - leave the logic for other
places.

### Keep your component modules generic

We always get asked "How do I know if I'm writing a good module?" We USED to
say "Well, does it work?" (and trust me, that was a BIG hurdle). Now, with
data separation models out there like Hiera, I have a couple of other questions
that I ask (you know, BEYOND asking if it compiles and actually installs the
thing it's supposed to install). The best way I've found to determine if your
module is 'generic enough' is if I asked you TODAY to give me your module,
would you give it to me, or would you be worried that there was some
company-specific data locked in there? If you have company-specific data in
your module, then you need to refactor the module, store the data in Hiera, and
make your module more generic/reusable. Also, does your module focus on installing
one piece of technology, or are you declaring packages for shared libraries
or other components (like gcc, apache, or other common components)? You're
not going to win any prizes for having the biggest, most monolithic module
out there. Rather, if your module is that large and that complex, you're
going to have a hell of a time debugging it. Err on the side of making your
modules smaller and more task-specific. So what if you end up needing to
declare 4 classes where you previously declared 1? In the roles and profiles
pattern we will show you in the next blog post, you can abstract that away
ANYHOW.

### Don't play the "what if" game

I've had more than a couple of gigs where the customer says something along the
lines of "What if we need to introduce FreeBSD/Solaris/etc... nodes into our
organization, shouldn't I account for them now?" This leads more than a few
people down a path of entirely too-complex modules that become bulky and
unwieldy. Yes, your modules should be formatted so that you can simply add
another case in your `::params` class for another OS's parameters, and yes,
your module should be formatted so that your `::install` or `::config`
class can handle another OS, but if you currently only manage Redhat, and
you've only EVER managed Redhat, then don't start adding Debian parameters
RIGHT NOW just because you're afraid you might inherit Ubuntu machines. The
goal of Puppet is to automate the tasks that eat up the MAJORITY of your time
so you can focus on the edge cases that really demand your time. If you can
eventually automate those edge cases, then AWESOME! Until then, don't spend
the majority of your time trying to automate the edge cases only to drown
under the weight of deadlines from simple work that you COULD have already
automated (but didn't, because you were so worried about the exceptions)!

### Store your modules in version control

This should go without saying, but your modules should be stored in version
control (a la git, svn, hg, whatever). We tend to prefer git due to its lightweight
branching and merging (most of our tooling and solutions will use git because
we're big git users), but you're free to use whatever you want. The bigger
question is HOW to store your modules in version control. There are usually
two schools of thought:

* One repository per module
* All modules in a single repository

Each model has its pros and cons, but we tend to recommend one module per
repository for the following reasons:

* Individual repos mean individual module development histories
* Most VCS solutions don't have per-folder ACLs for a single repositories;
having multiple repos allows per-module security settings.
* With the one-repository-per-module solution, modules you pull down from the
Forge (or Github) must be committed to your repo. Having multiple
repositories for each module allow you to keep everything separate

**NOTE:** This becomes important in the third blog post in the series when we
talk about moving changes to each Puppet Environment, but it's important to
introduce it NOW as a 'best practice'. If you use our recommended module/environment
solution, then one-module-per-repo is the best practice. If you DON'T use our
solution, then the single repository per for all modules will STILL work,
but you'll have to manage the above issues. Also note that even if you currently
have every module in a single repository, you can STILL use our solution in
part 3 of the series (you'll just need to perform a couple of steps to conform).

## Best practices are shit

In general, 'best practices' are only recommended if they fit into your organizational
workflow. The best and worst part of Puppet is that it's infinitely customizable,
so 'best practices' will invariably be left wanting for a certain subset of the
community. As always, take what I say under consideration; it's quite possible
that I could be entirely full of shit.

[puppetlabs]: http://www.puppetlabs.com
[rimodule]: http://www.devco.net/archives/2012/12/13/simple-puppet-module-structure-redux.php
[docsmodule]: http://docs.puppetlabs.com/guides/module_guides/bgtm.html
[almodule]: http://www.confreaks.com/videos/1651-puppetconf2012-puppet-modules-for-fun-and-profit
[hierablog]: http://garylarizza.com/blog/2013/12/08/when-to-hiera/
[forge]: http://forge.puppetlabs.com

