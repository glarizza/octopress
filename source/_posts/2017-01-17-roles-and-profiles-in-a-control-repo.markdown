---
layout: post
title: "Roles and Profiles in a Control Repo?"
date: 2017-01-17 11:14:36 -0800
comments: true
categories: ['r10k', 'puppet', 'workflow', 'roles', 'profiles', 'control repo']
---

In the past, the thing that got me to make a blog post was answering a question
more than once and not having a good source to point someone to after-the-fact.
As the docs at [docs.puppet.com](http://docs.puppet.com) have become more
comprehensive, I find that I'm wanting to write about things infrequently. But,
all it takes is a question or two from a customer to kick things in the
ass and remind me that there's still a *LOT* of tribal knowledge around Puppet
(let alone the greater community). It's with THAT theme that we talk about
Roles & Profiles, and the Control Repo.

[Like many things nowadays, there are official Puppet docs on the Control Repo.][control repo]
In a nutshell, the Control Repo is the repository that Puppet's Code Manager
(or R10k in the open source) uses to track Puppet Environments and the versions
of all Puppet modules within each Puppet Environment. On a greater scale, the
Control Repo is an organization's implementation of Puppet to the extent that
it can (and eventually will) fully represent your organization's
infrastructure. Changes to the Control Repo WILL incur changes to your Puppet
Master(s), and in most cases will also bubble down to your managed nodes
(i.e. if you're changing a profile that's being used by 1000 nodes, then that
change will be definitely change the file that's on each Puppet Master but will
also change the enforcement of Puppet on those 1000 nodes).

[Similarly, Roles & Profiles has its own official docs page!][roles and profiles]
As a recap, "Role & Profiles" is a design pattern (that's all!) that has been
employed by Puppet Users for several years as a way to make sense of wiring
up public Puppet modules with site-specific implementations and data. It allows
organizations to share common modules while also having the ability to add their
own customizations and implement a separate configuration management data layer
(i.e. Hiera).

Both the Control Repo and Roles & Profiles (R&P) have undergone several evolutions to
get them to the reliable state we know today, and they've had their shared
history: we've implemented Roles & Profiles both inside and outside the Control Repo...

## Roles and Profiles outside the Control Repo

Roles & Profiles were (was?) created before the Control Repo because the problem of
disentangling data from Puppet code was a greater priority than automating
code-consistency across multiple Puppet masters. When the idea of using a git
repo to implement dynamic Puppet Environments came along, the success of
being able to ensure module consistency across all your masters was pretty
landmark. The workflow, however, needed some work - there were a LOT of steps.
Git workflow automation loves it some hooks, and so the idea of a post-receive hook
that would immediately update a Puppet environment was a logical landing point.
The idea was that all modules would be listed and 'pinned' to their correct
version/tag/commit within `Puppetfile` that lived at the root of the Control
Repo. 'Roles' and 'Profiles' are Puppet modules, modules were already listed
in `Puppetfile`, so some customers listed them there initially. During a code
deploy, R10k/Code Manager would read that file, pull down all the modules at their
correct versions, and then move along. That entire workflow looked like this:

1. Create/Modify a Profile and push changes to the Profile module repo
2. Create a branch to the Control Repo and modify `Puppetfile` to target the new Profile changes
3. Push the Control Repo changes up to the remote (where the git webhook catches that change and deploys it to Puppet Masters)
4. Classify your node (if needed) and/or test the changes
5. If changes are necessary, go back to step 1 and repeat up to step 4
6. Once everything works, submit a Pull Request to the Control Repo

This workflow works regardless of whether a Role or Profile was changed, but
the biggest thing to understand is that ONLY the Control Repo has the git webhook
that will deploy code changes to your Puppet Masters, so if you want to trigger
a code deploy then you'll need to change the Control Repo and push that change
up (or have access to trigger R10k/Code Manager on the Puppet Master). This
resulted in a lot of 'dummy' changes that were necessary SOLELY to trigger a
code change. Conversely, changes to the Roles or Profiles module (they're separate)
don't get automatically replicated, so even if there's a small change to a
Profile you'll still need to either trigger R10k/Code Manager by hand or make
a small dummy commit to the Control Repo to trigger a code deploy.

As I said before, some customers implemented Roles & Profiles and the Control Repo
this way for awhile until it was realized that you could save steps by putting
both the Roles and Profiles module into the Control Repo itself...

## Roles and Profiles inside the Control Repo

Since the entire contents of the Control Repo are already cloned down to disk by
R10k/Code Manager, the idea came about to store the Roles and Profiles modules
in a special directory of the Control Repo (usually called 'site' which is short
for 'site-specific modules'), and then change `$modulepath` within Puppet to
look for the 'site' folder within every Puppet Environment's directory path as
another place for Puppet modules to live. This worked for two reasons:

1. It shortened the workflow (since changes to Roles and Profiles were done
   within the history of the Control Repo, there was no need to change the
   version 'pin' inside `Puppetfile` as a separate step)
2. Because Roles and Profiles are now within the Control Repo, changes made to
   Roles and Profiles will now trigger a code deploy

For the vast majority of customers, putting Roles & Profiles inside the Control
Repo made sense and kept the workflow shorter than it was before. It also had
the added benefit of turning the Control Repo into the important artifact that
it is today (thanks to the git webhook).

## Can/Should we also put other modules inside the Control Repo?

Once you add the site directory to `$modulepath`, it opens up that directory to
be used as a place for storing ANY Puppet modules. The question then remains:
should the site directory be used for anything else other than Roles and Profiles?

Maybe?

Just like Puppet, though, just because you CAN do things, doesn't immediately
mean you SHOULD. It's important to understand that the Control Repo is
fundamental to ensuring code consistency across multiple Puppet Masters. For
that reason, commits to the Control Repo should be scruitinized closely. If
you're a large team with many Puppet contributers and many Puppet masters, then
it's best to keep modules within their own git repositories so multiple team
members can work independenly and the Control Repo can be used to "tie it all
together" in the end. If you're the only Puppet contributor, you're using 80%
of your modules from the Puppet Forge, but you have 3 relatively-static modules
outside of Roles and Profiles that you've written specifically for your
organization and you want them in the site directory of the Control Repo then
you're probably fine. See the difference?

## Who owns what?

One of the biggest factors to influence where Puppet modules should be managed
is the split of which teams own which decisions. Usually, Puppet infrastructure
is owned by an internal operations team, which means that the Ops team is used
to making changes to the Control Repo. If Puppet usage is wide enough within
your organization it's common to find application teams who own specific
Profiles that are separate from the infrastructure. It's usually easier to
grant an outside team access to a separate repo than it is to try and restrict
access to a specific folder or even branch of an existing repository, and so
in that case it might make sense to make the Profile module its own repository.
If the people that own the Puppet infrastructure are the same people that
make changes to Puppet modules, then it doesn't really matter where Roles and
Profiles go.

For many organizations this is THE consideration that determines their choice,
but remember to build a workflow for today with the ability to adapt to
tomorrow.  If you have a single person outside the ops team contributing to
Puppet, it doesn't mean that you need to upend the workflow just for them.
Moving from something like having Roles & Profiles inside the Control Repo to
having them outside the Control Repo is an easy switch to implement (from
a technical standpoint), but the second you make that switch you're adding
steps to EVERYONE'S workflow and changing the location of the most commonly
used modules within Puppet. That's a heavy cost - don't do it without reason.

## So what are the OFFICIAL RECOMMENDATIONS THEN?!?!

We officially recommend you calm down with that punctuation. Beyond that, here it is:

* Put Roles & Profiles within the site directory of the Control Repo unless you
  have a specific reason NOT to.

Do you have multiple Puppet contributors and separate modules for EACH
INDIVIDUAL Profile?  Then you might want to have separate repos for each
Profile and put them in `Puppetfile` to keep the development history separate.
You ALSO might want to put each individual Profile module in the site directory
of the Control Repo and just run with it that way. The bottom line here would
be access: who can/should access Profiles, who can/should access the Control
Repo, are those people the same, and do you need to restrict access for some
reason? Start with doing it this way and change WHEN YOU HIT THAT COMPLEXITY!
Don't deviate because you 'anticipate something' - change when you're ready
for it and don't overarchitect early.

* If you're a smaller team with the same people who own the Puppet infrastructure
  as who own Puppet module development and you have a couple of small internal modules
  that don't change very often, AND putting them inside the 'site' folder of
  the Control Repo is easier for you than managing individual git repos, then by
  all means do it!

Whew that was a lot. Basically, yes, I've outlined a narrow case because usually
creating a new git repository is a very small cost for an organization. If
you're in an organization where that's NOT the case, then the site directory
solution might appeal to you. What you gain in simplicity you lose in access
and security, though, so consider that ahead of time. Finally, the biggest factor
HERE is that the same people own the infrastructure and module code, so you can
afford to make shortcuts.

* Have an internal Puppet Policy/Style Guide for where Puppet modules "go."

If you've had the conversation and made the decision, DOCUMENT IT! It's more
important to have an escalation path/policy for new Puppet users in your
organization to ensure consistency (the last thing you want to do is to keep
having this conversation every other month).

* Moving a module from the 'site' directory to its own repository is not
  difficult, but it does add workflow steps.

Remember that if a module doesn't live in the 'site' directory then it needs
to get 'pinned' in `Puppetfile`, and that adds an extra step anytime that
module needs updated within a Puppet Environment.


## Summary

First, if you've read this post it's probably because you're Googling for
material to support your cause (or someone cited this post as evidence to back
their position). You might have even skipped down here for "the answer." Guess
what - shit doesn't work like that! Storing Roles & Profiles (and/or other
Puppet modules) within the 'site' directory is an organizational choice based
on the workflow that best jives with an organization's existing developmental
cycle and ownership requirements. The costs/benefits for each choice boil down
to access, security, and saving time. The majority of the time putting Roles
& Profiles in the Control Repo saves time and keeps all organizational-specific
information in one place.  If you don't have a great reason to change that,
then don't.

[control repo]: https://docs.puppet.com/pe/latest/cmgmt_control_repo.html
[roles and profiles]: https://docs.puppet.com/pe/latest/r_n_p_intro.html

