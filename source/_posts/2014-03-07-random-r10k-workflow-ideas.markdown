---
layout: post
title: "On R10k and 'Environments'"
date: 2014-03-26 00:00:00 -0800
comments: true
categories: ['puppet', 'r10k', 'workflow', 'hiera']
---

There have been more than a couple of moments where I'm on-site with a customer
who asks a seemingly simple question and I've gone "Oh shit; that's a great
question and I've never thought of that..."  Usually that's followed by me
changing up the workflow and immediately regretting things I've done on prior
gigs. Some people call that 'agile'; I call it 'me not having the
forethought to consider conditions properly'. 

## 'Environment', like 'scaling', 'agent', and 'test', has many meanings

It's not a secret that we've made some shitty decisions in the past with regard
to naming things in Puppet (and anyone who asks me what `puppet agent -t`
stands for usually gets a heavy sigh, a shaken head, and an explanation emitted
in dulcet, apologetic tones). It's also very easy to conflate certain concepts
that unfortunately share very common labels (quick - what's the difference
between properties and parameters, and give me the lowdown on MCollective
agents versus Puppet agents!).

And then we have 'environments' + Hiera + R10k.

### Puppet 'environments'

Puppet has the concept of 'environments', which, to me, exist to provide a
means of compiling a catalog using different paths to Puppet modules on the
Puppet master. Using a Puppet environment is the same as saying "I made some
changes to my tomcat class, but I don't want to push it DIRECTLY to my production
machines yet because I don't drink Dos Equis. It would be great if I could stick
this code somewhere and have a couple of my nodes test how it works before
merging it in!"

Puppet environments suffer some 'seepage' issues,
[which you can read about here,][envbug] but do a reasonable job of quickly
testing out changes you've made to the Puppet DSL (as opposed to custom
plugins, as detailed in the bug). Puppet environments work well when you
need a pipeline for testing your Puppet code (again, when you're refactoring
or adding new functionality), and using them for that purpose is great.

### Internal 'environments'

What I consider 'internal environments' have a couple of names - sometimes
they're referred to as application or deployment gateways, sometimes as 'tiers', but
in general they're long-term groupings that machines/nodes are attached to
(usually for the purpose of phased-out application deployments). They
frequently have names such as 'dev', 'test', 'prod', 'qa', 'uat', and the
like.

For the purpose of distinguishing them from Puppet environments, I'm going to
refer to them as 'application tiers' or just 'tiers' because, fuck it, it's a
word.

### Making both of them work

The problems with having Puppet environments and application tiers are:

* Puppet environments are usually assigned to a node for short periods of time,
while application tiers are usually assigned to a node for the life of the node.
* Application tiers usually need different bits of data (i.e. NTP server
addresses, versions of packages, etc), while Puppet environments usually
use/involve differences to the Puppet DSL.
* Similarly to the first point, the goal of Puppet environments is to eventually
merge code differences into the main production Puppet environment. Application
tiers, however, may always have differences about them and never become unified.

You can see where this would be problematic - especially when you might want to
do things like use different Hiera values between different application tiers,
but you want to TEST out those values before applying them to all nodes in an
application tier. If you previously didn't have a way to separate Puppet
environments from application tiers, and you used R10k to generate Puppet
environments, you would have things like long-term branches in your repositories
that would make it difficult/annoying to manage.

**NOTE: This is all assuming you're managing component modules, Hiera data,
and Puppet environments using R10k.**

The first step in making both monikers work together is to have two separate
variables in Puppet - namely `$environment` for Puppet environments, and
something ELSE (say, `$tier`) for the application tier. The "something else" is
going to depend on how your workflow works. For example, do you have something
centrally that can correlate nodes to the tier in which they belong? If so, you
can write a custom fact that will query that service. If you don't have this
magical service, you can always just attach an application tier to a node in
your classification service (i.e. the Puppet Enterprise Console or Foreman).
Failing both of those, [you can look to external facts.][factsd] External Fact
support was introduced into Facter 1.7 (but Puppet Enterprise has supported
them through the standard lib for quite awhile). External facts give you the
ability to create a text file inside the facts.d directory in the format of:

```
tier=qa
location=portland
```

Facter will read this text file and store the values as facts for a Puppet run,
so `$tier` will be `qa` and `$location` will be `portland`. This is handy for
when you have arbitrary information that can't be easily discovered by the
node, but DOES need to be assigned for the node on a reasonably consistent
basis.  Usually these files are created during the provisioning process, but
can also be managed by Puppet.  At any rate, having `$environment` and `$tier`
available allow us to start to make decisions based on the values.

### Branch with $environment, Hiera with $tier

Like we said above, Puppet environments are frequently short-term assignments,
while application tiers are usually long-term residencies. Relating those back
to the R10k workflow: branches to the main puppet repo (containing the
`Puppetfile`) are usually short-lived, while data in Hiera is usually
longer-lived. It would then make sense that the name of the branches to the
main puppet repo would resolve to being `$environment` (and thus the Puppet
environment name), and `$tier` (and thus the application tier) would be used
in the Hiera hierarchy for lookups of values that would remain different across
application tiers (like package versions, credentials, and etc...).

Wins:

* Puppet environment names (like repository branch names) become relatively
meaningless and are the "means" to the end of getting Puppet code merged into
the PUPPET CODE's production branch (i.e. code that has been tested to work
across all application tiers)
* Puppet environments become short lived and thus have less opportunity to
deviate from the main production codebase
* Differences across application tiers are locked in one place (Hiera)
* Differences to Puppet DSL code (i.e. in Manifests) can be pushed up to the
profile level, and you have a fact (`$tier`) to catch those differences.

The ultimate reason why I'm writing about this is because I've seen people try
to incorporate both the Puppet environment and application tier into both the
environment name and/or the Hiera hierarchy. Many times, they run into all
kinds of unscalable issues (large hierarchies, many Puppet environments,
confusing testing paths to 'production'). I tend to prefer this workflow
choice, but, like everything I write about, take it and model it toward what
works for you (because what works now may not work 6 months from now).

## Thoughts?

Like I said before, I tend to discover new corner cases that change my mind
on things like this, so it's quite possible that this theory isn't the most
solid in the world. It HAS helped out some customers to clean up their code
and make for a cleaner pipeline, though, and that's always a good thing. Feel
free to comment below - I look forward to making the process better for all!


[envbug]: http://projects.puppetlabs.com/issues/12173
[factsd]: http://docs.puppetlabs.com/guides/custom_facts.html#external-facts
