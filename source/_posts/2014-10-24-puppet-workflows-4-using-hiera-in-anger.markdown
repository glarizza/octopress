---
layout: post
title: "Puppet Workflows 4: Using Hiera in anger"
date: 2014-10-24 09:13:49 -0400
comments: true
categories: ['puppet', 'workflow', 'hiera', 'hierarchy', 'create_resources', 'create_resources_is_bad_mmmkay']
---

Hiera. That thing nobody is REALLY quite sure how to say (FYI: It's pronounced
'hiera'), the tool that everyone says you should be using, and the tool that
will make you hate YAML syntax errors with a passion. It's a data/code
separation dream, (potentially) a debugging nightmare, and absolutely vital in
creating a Puppet workflow that scales better than your company's Wifi strategy
(FYI: your company's Wifi password just changed. Again. Because they're not
using certificates). I've already written a GOOD AMOUNT on why/how to use it,
but now I'm going to give you a couple of edge cases. Call them "best
practices" (and I'll cut you), but I like to call it "shit I learned
after using Hiera in anger." Here are a couple of the most popular questions
I hear, and my usual responses...


## "How should I setup my hierarchy?"

This is such a subjective question because it's specific to your organization
(because it's your data). I usually ask back "What are the things about your
nodes that are different, and when are they different?" Usually I hear something
back like "Well, nodes in this datacenter have different DNS settings" or
"Application servers in production use one version of java, and those in dev
use a different version" or "All machines in the dev environment in this datacenter
need to have a specific repository". All of these replies give me ideas to your
hierarchy.  When you think of Hiera as a giant conditional statment, you can
start seeing how your hierarchy could be laid out.  With the first response, we
know we need a `location` fact to determine where a node is, and then we can
have a hierarchy level for that location. The second response tells me we need
a level for the application tier (i.e. dev/test/prod).  The third response tells
me we need a level that combines both the location and the application tier. When
you add in that you should probably have a node-specific level at the top (for
overrides) and a default level at the bottom (or not: see the next section), I'm
starting to picture this:

{% codeblock lang:yaml %}
:hierarchy:
  - "nodes/%{::clientcert}"
  - "%{::location}/%{::applicationtier}"
  - "%{::location}/common"
  - "tier/%{::applicationtier}"
  - common
{% endcodeblock %}

Every time you have a need, you consider a level.  Now, obviously, it doesn't
mean that you NEED a level for every request (sometimes if it's an edge case
you can handle it in the profile or the role). There's a performance hit for
every level of your Hiera hierarchy, so ideally keep it minimal (or around
5 levels or so), but we're talking about flexibility here, and, if that's more
important than performance then you should go for it.

Next comes ordering. This one's SLIGHTLY easier - your hierarchy should read from
most-specific to least-specific. Note that when you specify an application tier
at a specific location that that it is MORE specific than just saying "all nodes in
this application tier." Sometimes you will have levels that might be hard to
define an order - such as location vs. application tier. You kinda just have to
go with your gut here. In many cases you may find that the data you put in those
two levels will be entirely different (location-based data may not ever overlap
with application-tier-specific data). Do remember than any time you change the
order of your hierarchy you're going to introduce the possibility that values
get flip/flopped.

If you look at level 3 of the hierarchy above, you'll see that I have 'common'
at the end. Some people like this syntax (where they put a 'common' file in a
folder that matches the fact they're checking against), and some people prefer
a filename matching the fact.  Do what makes you happy, but, in this case,
we can unify the location folder and just put the common file underneath the
application tier files.

Finally, DO MAKE USE OF FOLDERS!  For the love of god, this. Putting all files
in a single folder both makes that a BIG folder, but also introduces a namespace
collision (i.e. what if you have a location named 'dev' for example? Now you have
both an application tier and a location with the same name.  Oops).

How you setup your hierarchy is up to you, but this should hopefully give you
somewhere to start.



## An argument AGAINST {global,common}.yaml

I might as well drum up some controversy now...

When using Hiera, you need to define a hierarchy that Hiera uses in its search
for your data. Most often, it looks something like this:

{% codeblock lang:yaml hiera.yaml %}
---
:backends:
  - yaml
:yaml:
  :datadir: /etc/puppetlabs/puppet/hieradata
:hierarchy:
  - "nodes/%{::clientcert}"
  - "location/%{::location}"
  - "environment/%{::applicationtier}"
  - common
{% endcodeblock %}

Notice that little "common" at the end?  That means that, failing everything
else, it's going to look in `common.yaml` for a value.  This means that
`common.yaml` is a storage bin of default values.

Previously, you may have heard me rail against Hiera's optional second argument
and how I really don't like it.  Take this example:

{% codeblock lang:puppet %}
$foo = hiera('port', '80')
{% endcodeblock %}

Given this code, Hiera is going to look for a parameter called 'port' in its
hierarchy, and, if it doesn't find one in ANY of the levels, assign back a default
value of '80'.  I don't like using this second argument because:

1. If you forget to enter the 'port' parameter into the hierarchy, or typo it in the YAML file, Hiera will gladly assign the default value of '80' (which, unless you're checking for this, might sneak and get into production)
2. Where is the real 'default' value: the value in `common.yaml` or the optional second argument?

Okay, so we're resigned to never use the second argument.  Cool.  BUT, what
happens when you already have a value in `common.yaml` and you typo the name
of the parameter at a higher level (for example, you write "prot: 44" inside
a YAML file at a higher level)?  You're going to get the value in `common.yaml`,
and, again, unless you're watching for this, suddenly you get a default value
when you didn't expect it.

So, what is this value in `common.yaml` used for anyways? Well, it's a value
default for every node that needs it.  And where are we going to be getting
this value?  If you said "The Profile", you would be correct.  So what should
we do about this?  I say you hardcode the value right into the Profile (GASP!!)
I will talk about this more in the next section (because it's more appropriate
there), so just hold that thought for another section.

Until then, let me say that I've started to align myself with the idea that
`hiera.yaml` shouldn't have a "defaults" level.  The idea is that if there's
a value that every node should have, keep it visible and in the Profile. If
you need Hiera's dynamic lookup capability, THEN drop it into the hierarchy.
The downside of this philosophy is that you'll suddenly need to enter a value
in many different places (i.e. if suddenly you want to define a different port
in the 'dev' application tier than in all the other application tiers, then you
need to add the port parameter to EVERY ONE of your application tier files. If
you have 8 of these tiers, then you need to enter the value into all 8 files).
This is also less-than-ideal, but what are you optimizing for here?  You're
optimizing for early failure.  If Hiera can't find a value and it SHOULD, this
is an early failure at catalog compilation time. You'll know INSTANTLY that
there's a problem (instead of deploying a default value to production, for
example).

If you don't need this level of optimization, by all means keep the default
level.  If you don't want this much redundancy by entering values in multiple
files, by all means keep the default level.  I simply offer forth a solution
for those folks who are wanting a "fail-early, fail-often" approach in their
data lookup mechanism.

**NOTE**: You're never going to completely escape the idea of typos. Even
if you remove the "defaults" level in the hierarchy, you could still have the
case where you enter a port value in every file at the `$::applicationtier`
level, but maybe typo the parameter at a higher level.  In this case, Hiera
DOES return a value, but it's the `$::applicationtier` value.  This requires
"CONSTANT VIGILANCE!" to catch these cases, but the idea is that these edge
cases are much less than a defaults-level override, so you SHOULD be able to
spot it easier. No level of automation will totally remove human error (and
watch The Matrix if you don't believe me).


## Where to data - Hiera or Profile?

"Does this go right into the Profile or into Hiera?"  I get that question
repeatedly when I'm working with customers. It's a good question, and one of
the quickest ways to blow up your YAML files in Hiera. Here's the order I use
when deciding where to put data:

### Enter the data into the Profile itself

Yes, that's exactly what I said - put it in the Profile.  "BUT GARY!", you
should be thinking, "THAT'S EMBEDDING DATA INSIDE YOUR PUPPET CODE!"  To that
I say: "It depends on how you look at the problem and solution." The profile is
YOUR implementation.  It describes how YOU define the implementation of a piece
of technology in YOUR organization. As such, it's less about Puppet code and
more about pulling data and passing it TO the Puppet code. It's the glue-code
that grabs the data and wires it up to the model that uses it. How it grabs the
data is not really a big deal, so long as it grabs the RIGHT data - right? You
can choose to hardcode it into the Profile, or use Hiera, or use some other
magical data lookup mechanism - we don't really care (so long as the Profile
gathers the data and passes it to the correct Puppet class).

"BUT GARY! IT'S GOING TO MAKE IT HARD TO DEBUG!"  Not really. You're going to
have to open the Profile anyway to see what's going on (whether you pull the
data from Hiera or hardcode it in the Profile), right? And, arguably, the
Profile is legible...doing Hiera lookups gives you flexibility at a cost of
abstracting away how it got that bit of data (i.e. "It used Hiera"). For newer
users of Puppet, having the data in the Profile is easier to follow. So, in the
end, putting the data into the Profile itself is the least-flexible and most-visible
option...so consequently it should be the first available option. This option
is best for defaults that are applicable for ALL nodes.

Remember when I offered that crazy idea of not having a "defaults" level in the
hierarchy?  This is why: erring on the side of visibility in the Profile and
failing early with Hiera. It also gives you a logical starting point when
determining WHERE to put your data.  Moving along...

**PROS:**

* Data is clearly visible and legible in the profile (no need to open additional files)

**CONS:**

* Inability to redefine variables in Puppet DSL makes any settings constants by default (i.e. no overriding permitted)

### Enter the data into Hiera

If you find that you need to have different bits of data for different nodes
(i.e. a different version of Java in the dev tier instead of the prod tier),
then you can start to put the data into Hiera.

**NOTE**: If you subscribe to my "no defaults" philosophy above, you suddenly
need to enter the data into as many files as that level of the hierarchy has
(or more if you override at a higher level). Understand that you're not me,
and you DON'T NEED to subscribe to this "no defaults" view of the world. Yay
for being different!

Where to put the data is going to depend on your own needs - I'm trusting that
you can figure this part out :)

This answers that "where" question, but doesn't answer the "what" question...as
in "What data should I put into Hiera?"  For that, we have another section...

**PROS:**

* Flexibility in returning different values based on different conditions

**CONS:**

* Visibility - you must do a Hiera lookup to find the value (or open Hiera's YAML files)



## "What exactly goes into Hiera?"

If there were one question that, if answered incorrectly, could make or break
your Puppet deployment, this would be it. The greatest strength and weakness of
Hiera is its flexibility.  You can truly put almost anything in Hiera, and, when
combined with something like the create\_resources() function, you can create
your own YAML configuration language (tip: don't actually do this).

"But, seriously, what should go into Hiera, and what shouldn't?"

The important thing to consider here is the price you pay by putting data into
Hiera. You're gaining flexibility at a cost of visibility.  This means that you
can do things like enter values at all level of the hierarchy that can be
concatenated together with a single hiera\_array() call, BUT, you're losing the
visibility of having the data right in front of you (i.e. you need to open up
all the YAML files individually, or use the `hiera` binary to debug how you got
those values). Hiera is REALLY COOL until you have to debug why it grabbed (or
DIDN'T grab) a particular value.

Here's what I usually tell people about what should be put into Hiera:

* The exact data values that need to be different conditionally (i.e. a different ntp server for different sites, different java versions in dev/prod, a password hash, etc.)
* Dynamic data expressed in multiple levels of the hierarchy (i.e. a lookup for 'packages' that returns back an array of all the values that were found in all the levels of the hierarchy)
* Resources as a hash ONLY WHEN ABSOLUTELY NECESSARY

### Puppet manifest vs. create\_resources()

Bullets 1 and 2 above should be pretty straightforward - you either need to use
Hiera to grab a specific value or return back a list of ALL the values from ALL
the levels of the hierarchy. The point here is that Hiera should be returning
back only the minimal amount of data that is necessary (i.e. instead of
returning back a hash that contains the title of the resource, all the attributes
of the resource, and all the attribute values for that resource, just return
back a specific value that will be assigned to an attribute...like the password
hash itself for a user). This data lookup appears to be "magic" to new users of
Puppet - all they see is the magic phrase of "hiera" and a parameter to search
for - and so it becomes slightly confusing. It IS, however, easier to understand
that this magical phrase will return data, and that that data is going to be used
to set the value for an attribute. Consider this example:

{% codeblock lang:puppet %}
$password = hiera('garypassword')

user { 'gary':
  ensure   => present,
  uid      => '5001',
  gid      => 'gary',
  shell    => 'zsh',
  password => $password,
}
{% endcodeblock %}

This leads us to bullet 3, which is "the Hiera + create\_resources() solution."
This solution allows you to lookup data from within Hiera and pass it directly
to a function where Puppet creates the individual resources as if you had typed
them into a Puppet manifest itself. The previous example can be entered into
a Hiera YAML file like so:

{% codeblock lang:yaml sysadmins.yaml %}
users:
  gary:
    ensure: 'present'
    uid: '5001'
    gid: 'gary'
    shell: 'zsh'
    password: 'biglongpasswordhash'
{% endcodeblock %}

And then a resource can be created inside the Puppet DSL by doing the following:

{% codeblock lang:puppet %}
$users = hiera('users')
create_resources('users')
{% endcodeblock _%}

Both examples are functionally identical, except the first one only uses Hiera
to get the password hash value, whereas the second one grabs both the
attributes, and their values, for a specific resource. Imagine Puppet gives you
an error with the 'gary' user resource and you were using the latter example.
You grep your Puppet code looking for 'gary', but you won't find that user
resource in your Puppet manifest anywhere (because it's being created with the create\_resources() function).
You will instead have to know to go into Hiera's data directory, then the
correct datafile, and then look for the hash of values for the 'gary' user.

### Functional differences between the two approaches

Functionally, you COULD do this either way. When you come up with a solution
using create\_resources(), I challenge you to draw up another solution using
Puppet code in a Puppet manifest (however lengthy it may be) that queries Hiera
for ONLY the specific values necessary. Consider this example, but, instead,
you need to manage 500 users.
If you use create\_resources(), you would then need to add 500 more blocks to
the 'users' parameter in your Hiera datafiles.  That's a lot of YAML. And on
what level will you add these blocks? `prod.yaml`? `dev.yaml`? Are you using a
`common.yaml`? Your YAML files suddenly got huge, and the rest of your team
modifying them will not be so happy to scroll through 500 entries. Now consider
the first example using Puppet code. Your Puppet manifest suddenly grew, but it
didn't affect all the OTHER manifests out there: only this file. The Hiera YAML
files will still grow - but now 500 individual lines instead of 3000 lines in
the previous example. Okay, now which one is more LEGIBLE? I would argue that
the Puppet manifest is more legible, because I consider the Puppet DSL to be
very legible (again, subject to debate versus YAML). Moreover, when debugging,
you can stay inside Puppet files more often using Puppet manifests to define
your resources. Using create\_resources, you need to jump into Hiera more often.
That's a context shift, which adds more annoyance to debugging. Also, it
creates multiple "sources of truth." Suddenly you have the ability of entering
data in Hiera as well as entering it in the Puppet manifest, which may be clear
to YOU, but if you leave the company, or you get another person on your team,
they may choose to abuse the Hiera settings without knowing why.

Now consider an example that you might say is more tailored to create\_resources().
Say you have a defined type that sets up tomcat applications. This defined type
accepts things like a path to install the application, the application's package
name, the version, which tomcat installation to target, and etc. Now consider
that all application servers need application1, but only a couple
of servers need application2, and a very snowflake server needs application3 (in
this case, we're NOT saying that all applications are on all boxes and that their
data, like the version they're using, is different. We're actually saying that
different machines require entirely different applications).

Using Hiera + create\_resources() you could enter the resource for the
application1 at a low level, then, at a higher level, add the resource for
application2, and finally add the resource for application3 at the
node-specific level. In the end, you can do a hiera\_hash() lookup to discover
and concatenate all resources from all levels of the hierarchy and pipe that to
create\_resources.

How would you do this with Puppet code?  Well, I would create profiles for every
application, and either different roles for the different kinds of servers (i.e.
the snowflake machine gets its own role), or conditional checks inside the role
(i.e. if this node is at the London location, it gets these application profiles,
and etc...).

Now which is more legible? At this point, I'd still say that separate profiles
and conditional checks in roles (or sub-roles) are more legible - including
a class is a logical thing to follow, and conditionals inside Puppet code are
easy to follow. The create\_resources() solution just becomes magic. Suddenly,
applications are on the node. If you want to know where they came from, you
have to switch contexts and open Hiera data files or use the `hiera` binary
and do a debug run. If you're a small team that's been using Puppet forever,
then rock on and go for it. If you're just getting started, though, I'd shy
away.

### Final word on create\_resources?

{% codeblock %}
Some people, when confronted with a problem, think “I know, I'll use create_resources()."
Now they have two problems.
{% endcodeblock _%}

The create\_resources() function is often called the "PSE Swiss Army knife"
(or, Professional Services Engineer - the people who do what
I do and consult with our customers) because we like to break it out when we're
painted into a corner by customer requirements. It will work ANYWHERE, but, again,
at that cost of visibility. I am okay with someone using it so long as they
understand the cost of visibility and the potential debugging issues they'll hit.
I will always argue against using it, however, for those reasons. More code in
a Puppet manifest is not a bad thing...especially if it's reasonably legible
code that can be kept to a specific class. Consider the needs and experience
level of your team before using create\_resources() - if you don't have a good
reason for using it, simply don't.

### create_resources()

**PROS:**

* Dynamically iterate and create resources based on Hiera data
* Using Hiera's hash merging capability, you can functionally override resource values at higher levels of the hierarchy

**CONS:**

* Decreased visibility
* Becomes a second 'source of truth' to Puppet
* Can increase confusion about WHERE to manage resources
* When used too much, it creates a DSL to Puppet's DSL (DSLs all the way down)

### Puppet DSL + single Hiera lookup

**PROS:**

* More visible (sans the bit of data you're looking up)
* Using wrapper classes allows for flexibility and conditional inclusion of resources/classes

**CONS:**

* Very explicit - doesn't have the dynamic overriding capability like Hiera does




## Using Hiera as an ENC

One of the early "NEAT!" moments everyone has with Hiera is using it as an
External Node Classifier, or ENC. There is a function called `hiera_include()`
that allows you to include classes into the catalog as if you were to write
"include (classname)" in a Puppet manifest.  It works like this:

{% codeblock lang:yaml london.yaml %}
classes:
  - profiles::london::base
  - profiles::london::network
{% endcodeblock %}

{% codeblock lang:yaml dev.yaml %}
classes:
  - profiles::tomcat::application2
{% endcodeblock %}

{% codeblock lang:puppet site.pp%}
node default {
  hiera_include('classes')
}
{% endcodeblock _%}

Given the above example, the hiera\_include() function will search every level
of the hierarchy looking for a parameter called 'classes'. It returns
a concatenated list of classnames, which it then passes to Puppet's include()
function (in the end, Puppet will declare the profiles::london::base,
profiles::london::network, and profiles::tomcat::application2 classes). Puppet
puts the contents of these classes into the catalog, and away we go. This is
awesome because you can change the classification of a node conditionally
according to a Hiera lookup, and it's terrible because you can CHANGE THE
CLASSIFICATION OF A NODE CONDITIONALLY ACCORDING TO A HIERA LOOKUP!  This means
that anyone with access to the repo holding your Hiera data files can affect
changes to every node in Puppet just by modifying a magical key. It also means
that in order to see the classification for a node, you need to do a Hiera
lookup (i.e. you can't just open a file and see it).

Remember that WHOLE blog post about Roles and Profiles?  I do, because I wrote
the damn thing. [You can even go back and read it again, too, if you want to.][workflows2]
One of the core tenets of that article was that each node get classified with a
single role. If you adhere to that (and you should; it makes for a much more
logical Puppet deployment), a node really only ever needs to be classified
ONCE. You don't NEED this conditional classification behavior. It's one of those
"It seemed like a good idea at the time" moments that I assure you will pass.

Now, you CAN use Roles with hiera\_include() - simply create a Facter fact that
returns the node's role, add a level to the Hiera hierarchy for this role fact,
and in the role's YAML file in Hiera, simply do:

{% codeblock lang:yaml appserver.yaml %}
classes: role::application_server
{% endcodeblock _%}

Then you can use the same hiera\_include() call in the default node definition
in `site.pp`. The ONLY time I recommend this is if you don't already have some
other classification method. The downside of this method is that if your role
fact CHANGES, for some reason or another, classification immediately changes.
Facts are NOT secure - they can be overridden really easily. I don't like to
leave classification to an insecure method that anyone with root access on a
machine can change. Using an ENC or `site.pp` for classification means that the
node ABSOLUTELY CANNOT override its classification. It's the difference between
being authoritative and simply 'suggesting' a classification.

**PROS:**

* Dynamic classification: no need to maintain a site.pp file or group in the Console
* Fact-based: a node's classification can change immediately when its role fact does

**CONS:**

* Decreased visibility: need to do a Hiera lookup to determine classification
* Insecure: since facts are insecure and can be overridden, so can classification


## Data Bindings

In Puppet 3, we introduced the concept of 'data bindings' for parameterized classes,
which meant that Puppet now had another choice for gathering parmeter values.
Previously, the order Puppet would look to assign a value for parameters to
classes was:

1. A value passed to the class via the parameterized class syntax
2. A default value provided by the class

As of Puppet 3, this is the new parameter assignment order:

1. A value passed to the class via the parameterized class syntax
2. A Hiera lookup for *classname::parametername*
3. A default value provided by the class

Data bindings is meant to be pluggable to allow for ANY data backend, but,
as of this writing, there's currently only one: Hiera.  Because of this,
Puppet will now automatically do a Hiera lookup for every parameter to a
parameterized class that isn't explicitly passed a value via the parameterized
class syntax (which means that if you just do `include classname`, Puppet
will do a Hiera lookup for EVERY parameter defined to the "classname" class).

This is really cool because it means that you can just add *classname::parametername*
to your Hiera setup, and, as long as you're not EXPLICITLY passing that
parameter's value to the class, Puppet will do a lookup and find the value.

It's also completely transparent to you unless you know it's happening.

The issue here is that this is new functionality to Puppet, and it feels like
magic to me. You can make the argument and say "If you don't start using it,
Gary, people will never take to it," however I feel like this kind of magical
lookup in the background is always going to be a bad thing.

There's also another problem.  Consider a Hiera hierarchy that has 15 levels
(they exist, TRUST ME).  What happens if you don't define ANY parameters in
Hiera in the form of *classname::parametername* and simply want to rely on
the default values for every class?  Well, it means that Hiera is STILL going
to be triggered for every parameter to a class that isn't explicitly passed a
value.  That's a hell of a performance hit.  Fortunately, there's a way to
disable this lookup.  Simply add the following to the Puppet master's `puppet.conf`
file:

{% codeblock %}
data_binding_terminus = none
{% endcodeblock _%}

It's going to be up to how your team needs to work as to whether you use Hiera
data bindings or not. If you have a savvy team that feels they can debug these
lookups, then cool - use the hell out of it. I prefer to err on the side of an
explicit hiera() lookup for every value I'm querying, even if it's a lot of extra
lines of code. I prefer the visibility, especially for new members to your team.
For those people with large hierarchies, you may want to weigh the performance
hit.  Try to disable data bindings and see if your master is more performant. If
so, then explicit hiera() calls may actually buy you some rewards.

**PROS:**

* Adding parameters to Hiera in the style of *classname::parametername* will set parameterized class values automatically
* Simplified code - simply use the include() function everywhere (which is safer than the parameterized class syntax)

**CONS:**

* Lookup is completely transparent unless you know what's going on
* Debugging parameter values can be difficult (especially with typos or forgetting to set values in Hiera)
* Performance hit for values you want to be assigned the class default value


[workflows2]: http://bit.ly/puppetworkflows2
