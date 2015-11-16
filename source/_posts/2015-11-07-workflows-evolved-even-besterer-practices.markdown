---
layout: post
title: "Workflows Evolved: Even Besterer Practices"
date: 2015-11-16 09:00:24 -0600
comments: true
categories: ['r10k', 'puppet', 'workflow', 'environments', 'goddammitdavematthews']
---

It's nearly been two years since I posted the Puppet Workflow series and
several things have changed:

* R10k now ships with Puppet Enterprise and [there are docs for it!][r10kdocs]
* There's even a [`pe_r10k` module][per10k] that ships with Puppet Enterprise 2015.2.x and higher to configure R10k
* [Control repos are the standard and are popping up all over the place][pupcontrol]
* Most people are bundling Hiera data with their Control repo (unless they have a very good reason not to)
* Ditto for Roles and Profiles
* The one-role-per-node rule is a good start, but PE's rules-based classification engine allows us to relax that rule
* Roles still include Profiles, but conditional logic is allowed and recommended to keep Hiera hierarchy levels minimal
* 'Data' goes in Hiera, but the definition of 'data' changes between organizations
* There's now a (somewhat) defined path for whether 'data' is included in a profile or Hiera
* Automatic Parameter Lookup + Hiera...it's still hard to debug, but we're getting there
* I'm incredibly wary of taking Uber during peak travel times with rate multipliers

It's been awhile since I've had a good rant, so let's get right into it!

## Code Management with R10k

As of PE 3.8, R10k became bundled with Puppet Enterprise (PE) and was referred
to as "Code Management" which initially confused people because the only thing
about PE that was changed was that the R10k gem was preinstalled into PE's Ruby
installation.  The purpose of this act was twofold:

1. The Professional Services team was installing R10k in essentially EVERY services engagement, and so it made sense to ship R10k and thus officially support its installation
2. We've always had plans to keep the functionality that R10k provided but not NECESSARILY the tool-known-as-R10k, so calling the service it provided something OTHER than R10k would allow us to swap out the implementation underneath the hood while still being able to talk about the functionality it provided

Of course, if you didn't live inside Puppet Labs it's possible that you might not have gotten this
memo, but, hey: better late than never?

For various reasons, we also never initially shipped a PE-specific module to
configure R10k, so you ALSO had to either manually setup `r10k.yaml` or use
[Zack Smith's R10k module][zackr10k] to manage that file. Of course, that
module did all kinds of OTHER things (like installing the R10k gem, setting up
webhooks, and making my breakfast), which meant that if you used it with the
version of PE that shipped R10k, you had to be careful to use the version of
the module that didn't ALSO try to upgrade that gem on your system (and whoops
if the module actually upgraded the version of R10k that we shipped). This is
why that module is Puppet Approved but not an offical Puppet Labs module: it
does things that we would consider "unsupported" outside of a professional
services engagement (i.e. the webhook stuff). Finally, the path to
`r10k.yaml` was changed to `/etc/puppetlabs/r10k/r10k.yaml`, but, in its
absence, the old path of `/etc/r10k.yaml` would be used and a message would
be displayed to inform you of the new file path (in the case that both files
were present, the file at `/etc/puppetlabs/r10k/r10k.yaml` would win).

When PE version 2015.2.0 shipped (I'm still not used to these version numbers
either, folks), we FINALLY shipped a `pe_r10k` module with similar structure to
Zack's R10k module - this meant you could FINALLY setup R10k immediatly without
having to install additional Puppet modules. Even better(er), in PE 2015.2.2 we
expose [a couple of PE installer answer file questions][r10kanswers] that allow
you to configure R10k DURING INSTALL TIME - so now your servers could be
immediately bootstrapped with a single answers file (seriously, I know, it's
about time; I do this shit every week, you have no idea). It finally feels like
R10k has grown into the first-class citizen we all wanted it to be!

Which means it's time to dump it.

I kid. Mostly. The fact of the matter is that we're introducing a new service
to manage code within Puppet Enterprise, [and if you're interested in reading more about it, check out this blog post by Lindsay Smith about Code Manager.][cmblog]
For you, the consumer, the process will be the same: you have a control
repo, you push changes, a service is triggered on your Puppet masters, and code
is synchronized on the Puppet master. What WILL change is the setup of this tool
(there will still be PE installer answer file questions that allow you to configure
this service, don't fret, and you'll still be able to configure this service through
a Puppet module, but the name of said module and configuration files on disk
will probably be different. Welcome to IT).

Be on the lookout for this service, and, as always, check out the [PE docs site][docs] for
more information on the Code Management service.


## Control (repo) freak

With the explosion of R10k came the explosion of "Control Repos" all over the place.
Everyone had one, everyone had an opinion on what worked best, and, well, we didn't
really do a good job at offering a good startup control repo for you. Because of
that, [we recently posted a 'starter' control repo on Github][pupcontrol] in the Puppet Labs
namespace that could be used to get started with R10k. Yes, it's definitely long
overdue, but there it is! I use it on all engagements I do with new customers, so
you can guarantee it'll have the use of Puppet Labs' PS team behind it. If you've
not started with R10k yet (or if you have but you wanna see what kinda crazy shit
we're doing now), check it out. It's got great stuff in there like a config\_version
script to spit out the most recent commit of the current branch of the control repo
(read: also current Puppet environment) as the "Config Version" string that Puppet
prints out during every Puppet run ([see here for more info on this functionality][configversion]).
We're also slowly adding things like inital bootstrapping profiles that will do
things like configure R10k/Code Manager, manage the SSH key necessary to contact
the control repo (should you be using an internal git repository server and
also require an SSH key to access that repo), and so on. Star that repo and keep
checking back, especially around PE releases, to see if we've updated things in
a way that will help you out!


## "Just put it in the control repo"

Look, if there's one thing that my blog emphasizes (other than the fact that I've
got a hairpin trigger for cursing and an uncomfortable Harry Potter fetish) it's
that "best practices" are inversely related to the insecurities of the speaker.
Fortunately, I have no problem saying when I'm wrong. If you've got the time,
allow me my mea culpa moment. In the past I had recommended:

* Using a separate git repo for Hiera data
* Using separate git repos for Roles and Profiles
* The Dave Matthews Band

Time, experience, and the legalization of recreational marijuana in Oregon have
helped me see the error in my ways (though, look, #41 is a good goddamn song,
especially on the Dave & Tim Live at Luther College album), so allow me to provide
some insight into WHY I've reconsidered my message(s)...

### Hiera Data

In the past, I recommended a separate git repo for Hiera data along with
a separate entry in `r10k.yaml` that would allow R10k to clone the Hiera data repo
along the same vein as the control repo. The pro was that a separate Hiera data
repo would afford you different access rights to this repo as you would the
control repo (especially if different people needed different access to each
function). The con was that now the branch structure of your Hiera data repo
needed to EXACTLY MIRROR the structure of your control repo....even if certain
branches had EXACTLY THE SAME Hiera data and no changes were necessary.

Puppet has enough moving parts, why did we need to complicate this if most
people didn't care about access levels differing between the two repos? The
solution was to bundle the Hiera data inside the control repo all the way up
until you had a specific need to split it out. Truth be told both methods
work with Puppet, so the choice is up to you (read: I DON'T CARE WHICH METHOD
YOU USE OH MY GOD WILL YOU QUIT TRYING TO PICK A FIGHT WITH ME OVER THIS LOL) :)

Finally, there's an added benefit of putting this data inside the control repo,
and it's ALSO the reason for the next recommendation...

### Roles and Profiles

This is one that I actually fought when someone suggested it...I even started to
recommend that a customer NOT do the thing I'm about to recommend to you until they
very eloquently explained why they did it. In the end, they were right, and I'm
passing this tip on to you:  Unless you have a very specific reason NOT to,
put your 'roles' and 'profiles' modules in your control repo.

Here's the thing about the control repo - you can set a post-receive hook on
the repository (or setup a Jenkins/Bamboo/whatever job) that will update all your
Puppet masters whenever changes are pushed to the remote git repository (i.e.
your git repository server). This means that anytime the control repo is updated
your Puppet masters will be updated. That's why it's CALLED the control repo - it
effectively CONTROLS your Puppet masters.

Understanding THAT, think about when you want your Puppet masters updated? Well,
you usually want to update them when you're testing something out - you made a
change to a couple of modules, then a profile (and possibly also a role), and
now you wanna see if that code works on more than just your local laptop.
But the Puppet landscape has changed a bit as the Puppet Forge has matured - most
people are using modules off the Forge and are at least TRYING not to use their
own component modules. This means that changes to your infrastructure are being
controlled from within roles/profiles. But even IF you're one of those people
who aren't using the Forge or who have to update an internal component module,
you're probably not wanting to update all your Puppet masters every time you
update a component module. There's probably lots of tinkering there, and every
change isn't "update-worthy". Conversely, changes to your profiles probably
ARE "update-worthy": "Okay, let's pull this bit from Hiera, pass it as a parameter,
and now I'm ready to check it out on a couple of machines."

If your roles and profiles modules are separate from your control repo, you
end up having to push changes to, say, a class in the profiles module, then
updating the Puppetfile in the control repo, then trigger an R10k run/sync.
If things aren't correct, you end up changing the profile, pushing that change
to the profile repo, and THEN having to trigger an R10k run/sync (and if you
don't have SSH access to your masters, you have to make a dummy commit to the
control repo so it triggers an R10k run OR doing a curl to some endpoint that
will update your Puppet master for you). That last step is the thing that ends
up wasting a bit of your time: why do we need to push a profile and then manually
do an R10k run of we've established that roles and profiles will pretty much
ALWAYS be "update-worthy"? We don't. If you put the roles and profiles module
inside the control repo, then it will automatically update your Puppet masters
every time you make a change to one or the other. Bam - step saved. ALSO, if
you do this, you can take Roles/Profiles out of Puppetfile, which means you
no longer need to pin them! No more will you have to tie that module to a topic
branch during development time: just create a branch of the control repo and
go to town!  Wow, that saves even more time! I'm uncomfortable with this level
of excitement!

The one thing you WILL need to do is to update `environment.conf` so that it
knows to look for the roles/profiles modules in a different path from all the
other modules (because removing it from Puppetfile means that it will no longer
go to the same modulepath as every other module managed inside Puppetfile).
For the purposes of cleanliness, we usually end up putting both roles/profiles
inside a `site` folder in the control repo. If you do that, your modulepath
in `environment.conf` looks a little something like this:


{% codeblock %}
modulepath = site:modules:$basemodulepath
{% endcodeblock %}

This means that Puppet will look for modules first in the 'site' directory of
its current environment (this is the directory where we put roles/profiles),
and then inside the 'modules' directory (this is where modules managed in Puppetfile
are cloned by default), and then in $basemodulepath (i.e. modules common to all
environments and also modules that Puppet Enterprise ships).

LOOK, BEFORE YOU FREAK OUT, YES, SITE COMES FIRST HERE, AND OTHER PEOPLE HAVE
SITE COME SECOND! Basically, if you have roles/profiles in the 'site' directory
AND you manage to still have the module in Puppetfile, then the module in the 'site'
directory will win. Feel free to flip/flop that if you want.

**TL;AR: (yes, you already read all of this so it's futile) put roles/profiles
inside the site directory of the control repo to save you time, but also don't
do it if you have a specific reason not to...or if you like being contrarian.**

### Dave Matthews

The "Everyday" album was the "jump the shark" moment for the Dave Matthews band,
while the leaked "Lillywhite Sessions" that would largely make it to "Busted Stuff"
definitely indicated where the band wanted to go. They never recovered after that,
and, just like Boone's Farm 'wine', I stopped partaking in them.

Also, not ONCE did being able to play most every Dave Matthews song on the
acoustic guitar ever get me laid...though I can't tell exactly whose fault that
was. On second thought, that was probably me. Though Tim Reynolds is an absolute
beast of a musician; I'm still #teamtim.


## One role per node, until you don't want to

Why do we even make these rules if you're not gonna follow them? It's getting
awfully "Who's Line Is It Anyways?" up in here. Before PE 3.7, and its
rules-based classification engine, we recommended not assigning more than one
role to a node.  Why? Well, the Puppet Enterprise Console around that time
wasn't the best at tracking changes or providing authentication around tasks
like classification.  This meant if you tried to manage ALL of your
classification within the console you could have a hard time telling when
things changed or why. Fortunately, git provides you with this functionality.
Because of that, we (and when I say 'we' I mean 'everyone in the field trying
to design a Puppet workflow that not only made sense but also had some level of
accountability') tried to displace most classification tasks from the Console
into flat files that could be managed with git. This is largely the impetus for
Roles and Profiles when you think about it: Profiles connect Puppet to external
ata and give you a layer to express dependencies between multiple Puppet
classes, and Roles is a mechanism for boiling down classification to a single
unit.

Once we launched a new Node Classifier that had a rules-based classification
engine AND role-based authentication control, we became more comfortable
delegating some of these classification tasks BACK to the console. The Node
Classifier ALSO made it easy to click on a node and not only see what was
classified to that node, but also WHERE it got that bit of classification
from ("This node is getting the JBoss profile because it was put into the
App Servers nodegroup"). With that level of accountability, we could start
relaxing our "One Role Per Node™" mandate, OR eliminate the roles module
altogether (and use nodegroups in the Node Classifier in place of roles).

The goal has always been to err on the side of "debugability" (I like making words).
I will usually try to optimize a task for tracing errors later, because I've been
a sysadmin where the world is falling apart around you and you need to quickly
determine what caused this mess. Using one role per node makes sense if you
don't use a node classifier that gives you this flexibility, but MIGHT not if
you DO use a classifier that has some level of accountability.


## Roles, conditional logic, Hiera, and you

Over time as I've talked to people that ended up building Puppet workflows
based on the things I've written (which still feels batshit crazy to me,
by the way, since I've known myself for over 34 years), I've noticed that people
seem to take the things I say VERY LITERALLY. And to this I say: "You should
probably send me money via Paypal." Also - note that I'm writing these things
to address the 80% of people out there using/getting started with Puppet. You
don't HAVE to do what I say, especially if you have a good reason not to, and
you SHOULDN'T do what I say, especially if you're the one that's going to stay
with that organization forever and manage the entire Puppet deployment. For
everyone else out there, let's talk some more about roles.

The talking points around roles has always been "Roles include profiles; that's it."
Again, going back to the idea that roles exist to support classification, this
makes sense - you don't want to add resources at a very high level like a roles
class because, well, honestly, there's probably a better place for it, but any
logic added to simply classification is a win.

Consider an organization that has both Windows and Linux application servers.
The question of whether to have separate roles for Linux and Windows
application servers is always one of the first questions to be surfaced. At
a low level, everything you do in a Puppet manifest is solely for the
purpose of getting resources into the catalog (a JSON object containing
a list of all resource Puppet is to be managing ond their desired end-state).
Whether you have two different roles matters not to Puppet so long as the
right node gets the right catalog. For a Puppet developer writing code, having
two separate roles also might not matter (and, in reality, based on the amount
of code assigned to either role, it might be cleaner to have different roles
for each). For the person in charge of classifying nodes with their assigned
role, it's probably easier to have a single role (`roles::application_server`, for example)
that can be assigned to ALL application servers, and then logic inside the role
to determine whether this will be a Windows application server using IIS or
a Linux application server using JBoss (or, going further, a Linux application
server running Weblogic, or Websphere, or Tomcat, whatever). Like we mentioned
in the previous point, if you're using the "One role per node" philosophy, then
you probably want a single role with conditional logic to determine Windows/Linux,
and then determine Tomcat/JBoss, and so on. If you're using the Puppet Enterprise
Console's node classifier, and thus the rule-based engine, you can afford not
to care about the number of node groups you create because you can create a rule
to match for application servers, and then a rule to match on operating system,
and create as many rules as you want to dynamically discover and classify nodes
on the fly.

The point here is that the PURPOSE of the Role is to aid classification, and
the focus on creating a role is to start small, use conditional logic to
determine which profiles to include, and then simply include them. If that
conditional logic uses Facter facts, awesome. If you need to look at a variable
coming from the Console to do the job, fine - go for it! But if you're using
the Role as a substitute for a Profile (i.e. data lookups, declaring classes,
even declaring resources), then you're probably going down a path that's gonna
make it confusing for people follow what's going on.

Bottom line: technology-agnostic roles that utilize conditional logic around
including profiles is a win, but keep tasks like declaring resources and
component modules to Profiles. Doing this provides a top-down path for
debugging and a cleaner overall Puppet codebase.


## What the hell is 'Data' anyhow?

This point has single-handedly caused more people to come up and argue with me.
I'm not kidding. I shit you not, I've had people legitimately \*SCREAM\* at me
about how wrong I was with my opinions here. The cool thing is that people LOVE
the idea of Hiera - it lets you keep the business-specific data out of your
Puppet manifests, it's expressed in YAML and not the Puppet DSL, and when it
works, it's magical.

The problem is that it's fucking magical. Seriously.

So what IS a good use of Hiera? Anytime you have a bit of data that is subject
to override (for example: the classical NTP problem where everyone should use
the generic company NTP server, except nodes at this location should use a
different NTP server, and this particular node should use ITSELF as its NTP
server), that bit of data goes into Hiera (and by 'that bit of data', I mean
'the value of the NTP server' or 'the NTP server's FQDN'), which would look
SOMETHING like this:

{% codeblock lang:yaml %}
ntpserver: pool.ntp.org
{% endcodeblock %}

What does NOT go into Hiera is a hash-based representation of the Puppet
resource that would then be passed to create\_resources() and used to create
the resource in the catalog...which would look something like this:

{% codeblock lang:yaml %}
ntpfiles:
  '/etc/ntp/ntpd.conf':
    ensure: file
    owner:  0
    group:  0
    mode:   0644
    source: 'puppet:///modules/ntp/ntpd.conf'
{% endcodeblock %}

...which would then be passed into Puppet like this:

{% codeblock lang:puppet %}
create_resources('file', hiera_hash('ntpfiles))
{% endcodeblock %}

Yes, this is an exaggeration based on a very narrow use case, but what I'm trying
to highlight is that the 'data' bit in all that above mess is SOLELY an FQDN,
and everything else is arguably the "Model", or your Puppet code.

Organizations LOVE that you can put as much "stuff" into Hiera as you want and
then Puppet can call Hiera, create resources based on what it tells you, and
merrily be on your way. Well, they "love" it until it doesn't work or does
something unexpected, and then debugging Hiera is a right bastard.

Understand that the problem I have would be with unexpected Hiera behavior. If
you're skilled in the ways of the Hiera and its (sometimes cloudy) interaction
with Puppet, then by ALL means use it for whatever ya like. BUT, if you're
still new to Puppet, then you may have a very loose mental map for how Hiera
works and where it interacts with Puppet...and nobody should have to have that
advanced level of knowledge just to debug the damn thing.

The Hiera + create\_resources() use above is of particular nastiness simply
because it turns your Hiera YAML files into a potential mechanized weapon of Puppet
destruction.  If I know that you're doing this under the hood, I could
POTENTIALLY slip data into Hiera that would end up creating resources on a node
to do what I want. Frequently Puppet code is more heavily scrutinized than
Hiera data, and I could see something like this getting overlooked (especially
if you don't have a ton of testing around your Puppet code before it gets
deployed).

The REASON why create\_resources() was created was because Puppet lacked the
ability to do things like recursion and loops inside the DSL, and sometimes
you WANT to automate very repeated tasks. Consider the case where you truly
DON'T know how many of something is going to be on a node ahead of time - maybe
you're using VMware vRO/vRA and someone is building a node on-the-fly with
the web GUI. For every checkbox someone ticks there will be another application
to be installed, or another series of firewall rules, or SOMETHING like that.
You can choose to model these individually with profiles, OR, if the task is
repetitive, you can accept their choices as data and feed it back into Puppet
like a defined resource type. In fact, most use-cases for Hiera + create\_resources()
is passing data into a defined resource type. As of Puppet 4.x.x, we have
looping constructs inside the DSL, so we can finally AUTOMATE these tasks
without having to use an extra function (of course, in THIS use case, whether
you use recursion/looping in the DSL or create\_resources() matters not - you
get the same thing in the end).

For one last point, the Puppet DSL is still pretty easy to read (as of right now),
and most people can follow what's going on even if they're NOT PuppEdumicated.
Having 10 resource declarations in a row seems like a pain in the ass to write
when you're doing it, but READING it makes sense. Later on, if you need to know
what's going on with this profile, you can scan it and see exactly what's there.
If you start slipping lots of data into Hiera and looping logic into the DSL,
you're gonna force the person who manages Puppet to go back and forth between
reading Hiera code, then back to Puppet code, then back to the node, and so on.
Again, it's totally possible to do now, and frequently NECESSARY when you have
a more complex deployment and well-trained Puppet administrators, but initially
it's possible to build your own DSL to Puppet by slipping things into Hiera and
running away laughing.

So when do I put this 'data' into the Profile and when is a good time to put it
into Hiera?  I'm glad you asked...


## A path to Hiera data

These last two points I've written about before. I may be repeating myself, but
bytes are cheap. Like I wrote above (and before), putting data directly into a
Profile is the easiest and most legible way of providing "external data" into
Puppet. Yes, you'll argue, putting the data into a Profile, which is Puppet code,
is ARGUABLY NOT being very "external" about it. In my opinion it is - your Profile
is YOUR IMPLEMENTATION of a technology stack, and thus isn't going to be shared
outside your organization. I consider that external to all the component modules
out there, but, again, potato/potato. I recommend STARTING HERE when you're getting
started with Puppet. Hiera comes in when you have a very clear-cut need for
overriding data (a la: this NTP server everywhere, except here and here). The second
you might need to have different data, you can either start building conditional logic
inside the Profile, OR use the conditional logic that Hiera provides.

So - which do you use?

The point of Hiera is to solve 80% or better of all conditional choices in your
organization. Consider this data organization model:

* Everyone shares most of the same data items
* San Francisco/London do their own things sometimes
* Application tiers get their own level for dev/test/qa/prod-specific overrides
* Combinations of tiers/locations/and business units want their own overrides
* Node specific data is the most specific (and least-used) level

If you're providing some data to Puppet that follows this model, then cool
- use Hiera. What about specific "exceptions" that don't fit this model? Do you
try to create specialized layers in Hiera just for these exceptions? Certain
organizations absolutely do - I see it all the time. What you find is that
certain layers in Hiera go together (this location/tier/business\_unit level
goes right above location/tier, which goes right above location), and we
start referring to those coupled layers as "Chains". Chains are usually tied
to some specific need (deploying applications, for example). Sometimes you
create a chain just to solve a VERY SPECIFIC hard problem (populating
`/etc/sudoers` in large organizations, for example).

The question is - do I create another "Chain" of layers in the hierarchy
solely because deploying `sudoers` is hard, or do I throw a couple of case
statements into the `sudoers` profile and keep it out of Hiera altogether?

My answer is to start with conditional logic in the `sudoers` profile and break
it out into Hiera if you see that "Chain" being needed elsewhere. Why? Because, like
I've said many times before, debugging Hiera kinda sucks right now - there's no
way currently to get a dump of all variables and parameters for a particular node
and determine which were set by Hiera, which were set with variables in the DSL, which
came out of the console, and so on. If we HAD that tool, I'd be all about using
it and polluting your hierarchy all day long (I expand upon this slightly in the
next point about the Automatic Parameter Lookup + Hiera).

Bottom line: Start with the data in the Profile, then move it to Hiera when you
need to override. Start with conditional logic in the Profile, then create a
"Chain" in the Hierarchy if you need to use it in more than one place.


## Hiera, APL, Refactoring, WTF

Like I said, I've written about this before. I like the Automatic Parameter
Lookup functionality in Puppet - it's ace. I like Hiera. But if you don't know
how it works, or that it exists, it feels too much like Magic. There are certain
things in the product that can ONLY be set by putting data inside Hiera and running
Puppet, and that is truly an awesome thing: just tell a customer "drop this bit
of data somewhere in Hiera, run Puppet, and you're all set." But, again, if you
need to know how a particular line got into a particular config file on your
node, and it was set with the APL, then you've got some digging to do.

There's still no tool, like I mentioned in the last item, to give me full
introspection into all variables/parameters set for a node and that
variable/parameter's origin.  Part of the reason as to WHY this tool doesn't
exist is because the internals of Puppet don't necessarily make it easy for you
to determine where a parameter/variable was set.  That's OUR problem, and
I feel like we're slowly making progress on marking these things internally so
we can expose them to our customers. Until then, you have to trace through code
and Hiera data.

I know the second I publish and tweet about this, I'm gonna get a message from
R.I. Pienaar saying that I've crazy for NOT pushing people toward using Hiera
more with the Automatic Parameter Lookup, because the more we use it, the faster
we can move away from things like params classes, and profiles, and everything
else, but the reality is I'm ALL ABOUT PEOPLE using it if they know how it works.
I'm ACTUALLY fucking happy that it works well for you - please continue to use
it and do awesome Puppet things. I only recommend to people who are getting
started to NOT USE it FIRST, and then, when you understand how it would help
you by clocking some hours of Puppet code writing and debugging, do some refactoring
and move to it!

Yes, refactoring is involved.

Look, refactoring is a way of life. You're gonna re-tool your Puppet code for
the purposes of legibility, or efficiency, or any of the many other reasons why
you refactor code - it's unavoidable. Also, if I come into your org and setup
Puppet for the most efficient use-case, and then I leave that into your
relatively-new-to-Puppet hands, it's probably not gonna be the best situation
because you won't have known WHY I made the decisions I did (and, even if I
document them, you might have gaps of knowledge that would help you understand
the problems I'm helping you avoid).

Sometimes hitting the problem so you have first-hand knowledge of why you need
to avoid it in the future isn't the WORST thing in the world.

To move to any configuration management system means you're gonna be
refactoring.  Embrace it. Start small, get things working, then clean it up.
Don't try to build the "fortress of sysadmin perfection" with your first bit of
Puppet code - just get shit done! Allow yourself time during the month simply
to unwind some misgivings you realize after-the fact, and definitely seek
advice before doing something you feel might be particularly complex or
overarching, but getting shit done is gonna trump "not working" any day (or
whatever the manager-y buzzspeak is this week).


Bottom Line: APL if you understand it, start small, get shit done, refactor, repeat


## Hopefully this leads to more posts

Holy shit, you're still reading?! Ohh, you skimmed down this far to see how long
this post was gonna be - got it. Either way, I'm glad I finally got this out there.
It's been months, yes, but that doesn't mean I haven't been writing. We've been
doing lots of internal work to try and get more official docs out to you and
less of "Go read Gary's blog!" You'll notice [R10k has some official docs, right?!][r10kdocs]
Yeah, that's awesome! We want more of that. BUT, there's still going to be times
where I feel like what I'm gonna say isn't necessarily the "party line", and that's
what this blog is about.

Thanks to everyone at Puppetconf and beyond who approached me and told me how
much they love what I write. I'm gonna be humble as fuck in person, but I really
do get excited whenever someone says that. It's also crazy as hell when someone
from Wal-mart approaches you and says they built part of their deployment based
on the shit you wrote. From a guy who came from a town in Ohio with a population
of less than 8000 people, it's crazy to see where you're "recognized."

So thank you, again, for all the support.



And sorry, Dave Matthews - it's not you, it's me. Actually, that's a lie; it was you.


[r10kdocs]: http://docs.puppetlabs.com/pe/latest/r10k.html
[per10k]: http://docs.puppetlabs.com/pe/latest/r10k_config_console.html
[pupcontrol]: https://github.com/puppetlabs/control-repo
[zackr10k]: https://github.com/acidprime/r10k
[r10kanswers]: http://docs.puppetlabs.com/pe/latest/r10k_config_answers.html
[docs]: http://docs.puppetlabs.com
[configversion]: https://docs.puppetlabs.com/puppet/latest/reference/config_file_environment.html#configversion
[cmblog]: https://puppetlabs.com/blog/managing-infrastructure-as-code-now-easier-than-ever
