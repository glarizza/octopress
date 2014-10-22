---
layout: post
title: "On Dependencies and Order"
date: 2014-10-19 14:09:53 +0100
comments: true
categories: ['puppet', 'order', 'dependencies', 'MOAR', "seriously-don't-use-manifest-ordering-as-a-replacement-for-dependencies"]
---


This blog post was born out of a number of conversations that I've had about
Puppet, its dependency model, and why 'ordering' is not necessarily the way to
think about dependencies when writing Puppet manifests. Like most everything on
this site, I'm getting it down in a file so I don't have to repeat this all over
again the next time someone asks. Instead, I can point them to this page (and,
when they don't actually **READ** this page, I can end up explaining everything
I've written here anyways...).

Before we go any further, let me define a couple of terms:

{% codeblock %}
dependencies     - In a nutshell, what happens when you use the metaparameters of
                   'before', 'require', 'subscribe' or 'notify' on resources in a
                   Puppet manifest: it's a chain of resources that are to be
                   evaluted in a specific order every time Puppet runs. Any failure
                   of a resource in this chain stops Puppet from evaluating the
                   remaining resources in the chain.

evaluate         - When Puppet determines the 'is' value (or current state) of a
                   resource (i.e. for package resources, "is the package installed?")

remediate        - When Puppet determines that the 'is' value (or current state of
                   the resource) is different from the 'should' value (or the value
                   entered into the Puppet manifest...the way the resource SHOULD
                   end up looking on the system) and Puppet needs to make a change.

declarative(ish) - When I use the word 'declarative(ish)', I mean that the order
                   by which Puppet evaluates resources that do not contain dependencies
                   does not have a set procedure/order. The way Puppet EVALUATES
                   resources does not have a set procedure/order, but the order
                   that Puppet reads/parses manifest files IS from top-to-bottom
                   (which is why variables in Puppet manifests need to be declared
                   before they can be used).
{% endcodeblock %}


## Why Puppet doesn't care about execution order (until it does)

The biggest shock to the system when getting started with a declarative (ish)
configuration management tool like Puppet is understanding that Puppet describes
the end-state of the machine, and NOT the order that it's (Puppet) going to
take you to that state. To Puppet, the order that it chooses to affect change
in any resource (be it a file to be corrected, a package to be installed, or
any other resource type) is entirely arbitrary because resources that have no
relationship to another resource shouldn't CARE about the order in which they're
evaluated and remediated.

For example, imagine Puppet is going to create both `/etc/sudoers` and update
the system's authorized keys file to enter all the sysadmins' SSH keys. Which
one should it do first? In an imperative system like shell scripts or
a runbook-style system, you are forced to choose an order. So I ask again,
which one goes first? If you try to update the `sudoers` file in your script
first, and there's a problem with that update, then the script fails and the
SSH keys aren't installed. If you switch the order and there's a problem with
the SSH keys, then you can't `sudo` up because the `sudoers` file hasn't been
touched.

Because of this, Puppet has always taken the stance that if there are failures,
we want to get as much of the system into a working state as possible (i.e. any
resources that don't depend upon the failing resource are going to still be
evaluated, or 'inspected', and remediated, or 'changed if need be'). There are
definitely philosophical differences here: the argument can be made that if there's
a failure somewhere, the system is bad and you should cast it off until you've
fixed whatever the problem is (or the part of the code causing the problem). In
virtualized or 'cloud' environments where everything is automated, this is just
fine, but in environments without complete and full automation, sometimes you
have to fix and deal with what you have. Puppet "believes in your system", which
is borderline marketing-doubletalk for "alert you of errors and give you time
to fix the damn thing and do another Puppet run without having to spin up a whole
new system."

Once you know WHY Puppet takes the stance it does, you realize that Puppet does
not give two shits about the order of resources without dependencies. If you
write perfect Puppet code, you're fine. But the majority of the
known-good-world does not do that. In fact, most of us write shit code. Which
was the problem...


## The history of Puppet's ordering choices

###  'Random' random order
In the early days, the only resources that were guaranteed to have a consistent
order were those resources with dependencies (i.e. as I stated above, resources
that used the 'before', 'require', 'subscribe', or 'notify' metaparameters to
establish an evaluation order). Every other resource was evaluted at random
every time that Puppet ran...which meant that you could run Puppet ten times
and, theoretically, resources without dependencies could be evaluated in
a different order between every Puppet run (we call this non-deterministic
ordering). This made things REALLY hard to debug.  Take the case where you had
a catalog of thousands of resources but you forgot a SINGLE dependency between
a couple of file resources. If you roll that change out to 1000 nodes, you
might have 10 or less of them fail (because Puppet chose an evaluation order
that ordered these two resources incorrectly). Imagine trying to figure out
what happened and replicate the problem. You could waste lots of time just
trying to REPLICATE the issue, even if it was a small fix like this.

**PROS**:

* IS there a pro here?

**CONS**:

* Ordering could change between runs, and thus it was very hard to debug missing dependencies

Philosophically, we were correct: resources that are to be evaluated in a certain
order require dependencies. Practically, we were creating more work for ourselves.

Incidentally, I'd heard that Adam Jacob, who created Chef, had cited this reason
as one of the main motivators for creating Chef. I'd heard that as a Puppet
consultant, he would run into these buried dependency errors and want to flip
tables. Even if it's not a true STORY, it was absolutely true for tables where
I used to work...


### Title-hash, 'Predictable' random order

Cut to Puppet version 2.7 where we introduced deterministic ordering with
'title-hash' ordering. In a nutshell, resources that didn't have dependencies
would still be executed in a random order, but the order Puppet chose could be
replicated (it created a SHA1 hash based on the titles of the resources without
dependencies, and ordered the hashes alphabetically). This meant that if you
tested out a catalog on a node, and then ran that same catalog on 1000 other
nodes, Puppet would choose the same order for all 1000 of the nodes. This
gave you the ability to actually TEST whether your changes would successfully
run in production. If you omitted a dependency, but Puppet managed to pick the
correct evaluation order, you STILL had a missing dependency, but you didn't
care about it because the code worked. The next change you made to the catalog
(by adding or removing resources), the order might change, but you would
discover and fix the dependency at that time.

**PROS**:

* 'Predictable' and repeatable order made testing possible

**CONS**:

* Easy to miss dependency omissions if Puppet chose the right order (but do you really care?)


###  Manifest ordering, the 'bath salts' of ordering

Title-hash ordering seemed like the best of both worlds - being opinionated about
resource dependencies but also giving sysadmins a reliable, and repeatable, way
to test evaluation order before it's pushed out to production.

Buuuuuuuuuut, y'all JUST weren't happy enough, were you?

When you move from an imperative solution like scripts to a declarative(ish)
solution like Puppet, it is absolutely a new way to think about modeling your
system. Frequently we heard that people were having issues with Puppet because
the order that resources shows up in a Puppet master WASN'T the order that Puppet
would evaluate the resources. I just dropped a LOT of words explaining why this
isn't the case, but who really has the time to read up on all of this? People
were dismissing Puppet too quickly because their expectations of how the tool
worked didn't align with reality. The assumption, then, was to align these
expectations in the hopes that people wouldn't dismiss Puppet so quickly.

[Eric Sorenson wrote a blog post on our thesis and experimentation][moarblog]
around manifest ordering that is worth a read (and, incidentally, is shorter
than this damn post), but the short version is that we tested this theory out
and determined that Manifest Ordering would help new users to Puppet. Because
of this work, we created a feature called 'Manifest Ordering' that stated that
resources that DID NOT HAVE DEPENDENCIES would be evaluated by Puppet in the
order that they showed up in the Puppet manifest (when read top to bottom). If
a resource truly does not have any dependencies, then you honestly should not
care one bit what order it's evaluated (because it doesn't matter).  Manifest
Ordering made ordering of resources without dependencies VERY predictable.

But....

This doesn't mean I think it's the best thing in the world. In fact, I'm really
wary of how I feel people will come to use Manifest Ordering. There's a reason
I called it the "bath salts of ordering" - because a little bit of it, when
used correctly, can be a lovely thing, but too much of it, used in unintended
circumstances, leads to hypothermia, paranoia, and the desire to gnaw someone
else's face off. We were/are giving you a way to bypass our dependency model by
using the mental-model you had with scripts, but ALSO telling you NOT to rely
on that mental-model (and instead set dependencies explicitly using metaparameters).

Seriously, what could go wrong?

Manifest Ordering is not a substitution for setting dependencies - that IS NOT
what it was created for. **Puppet Labs still maintains that you should use
dependencies to order resources and NOT simply rely on Manifest Ordering as
a form of setting dependencies!** Again, the problem is that you need to KNOW
this...and if Manifest Ordering allows you to keep the same imperative
"mindset" inside a declarative(ish) language, then eventually you're going to
experience pain (if not today, but possibly later when you actually try to
refactor code, or share code, or use this code on a system that ISN'T using
Manifest Ordering). A declarative(ish) language like Puppet requires seeing
your systems according to the way their end-state will look and worrying about
WHAT the system will look like, and not necessarily HOW it will get there. Any
shortcut to understanding this process means you're going to miss key bits of
what makes Puppet a good tool for modeling this state.


**PROS:**

* Evaluation order of resources without dependencies is absolutely predictable

**CONS:**

* If used as a substitution for setting dependencies, then refactoring code (moving around the order in which resources show up in a manifest) means changing the evaluation order


## What should I actually take from this?

Okay, here's a list of things you SHOULD be doing if you don't want to create
a problem for future-you or future-organization:

* Use dependency metaparameters like 'before', 'require', 'notify', and 'subscribe' if resources in a catalog NEED to be evaluated in a particular order
* Do not use Manifest Ordering as a substitute for explicitly setting dependencies (disable it if this is too tempting)
* Use Roles and Profiles for a logical module layout (see: [http://bit.ly/puppetworkflows2](http://bit.ly/puppetworkflows2) for information on Roles and Profiles)
* Order individual components inside the Profile
* Order Profiles (if necessary) inside the Role

And, seriously, trust us with the explicit dependencies. It seems like a giant
pain in the ass initially, but you're ultimately documenting your infrastructure,
and a dependency (or, saying 'this thing MUST come before that thing') is a pretty
important decision. There's a REASON behind it - treat it with some more weight
other than having one line come before another line, ya know? The extra time
right now is absolutely going to buy you the time you spend at home with your
kids (and by 'kids', I mean 'XBox').

And don't use bath salts, folks.

[moarblog]: http://puppetlabs.com/blog/introducing-manifest-ordered-resources

