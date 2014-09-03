---
layout: post
title: "Managing a blog is insane; Octopress FTW!"
date: 2013-01-17
comments: true
categories: ['blog', 'octopress', 'git']
---

Masochists are everywhere. Managing a system by hand appeals to a certain subset
of the population, but I joined [Puppet Labs][pl] because
I was a fan of automating the shit out of menial tasks. I have no burning desire
to handcraft an artisanal blog made out of organic bits. I tried web development
at one point in my life and learned an important lesson:

I am not a web developer

What I DO want is to have a platform to share information and code I've
accumulated along the way that:

1. Doesn't look like hell
2. Doesn't take forever to update
3. Doesn't require me being online to write a post
4. Allows me to post code that's syntactically highlighted and easy to copy/paste
5. Accepts [Markdown][markdown]
6. Fits into my DVCS workflow
7. Is free - because screw paying for A BLOG


## Seriously, is this too much to ask?

Originally I waded the waters of [Posterous][gl] but
found that not only did it have awkward [Markdown][markdown] syntax, but staging of
blog posts was cumbersome.  Also, while there WAS [gist][gist] integration,
the code you posted looked like crap. That sucks because most of what I post has
code attached.

Others have sold their soul for Wordpress/Drupal/Bumblefuck CMS (whether hosted
or unhosted platforms), but it still felt like too much process and not enough
action for my taste.


## Github to the rescue...again

The answer, it turns out, was right in front of my face the whole time.
[Github Pages][ghp] has always been available to host web content or [Jekyll
sites][jekyll] - I'm just late to the damn party.

[{% img left /images/blog_blog/left.png 300 300 %}](http://glarizza.github.com/images/blog_blog/left.png)

Hosting static web content was out; like I said before, I don't really want to
'manage' this thing. Jekyll, however, intrigued me. Jekyll is essentially
a static site generator that allows you to throw Markdown at it and get static
content in the end. There are even [Jekyll
bootstrap](http://jekyllbootstrap.com/) projects aimed at making it (even more)
stupid simple. I tried plain vanilla Jekyll and realized that I didn't really
want/need that level of control (again, I'm not into web development).
I pulled down Jekyll Bootstrap, but this time it felt a little TOO
'template-y'. Next, I pulled down an awesome Jekyll template called [Left by
Zach Holman](https://github.com/holman/left) (seen, conveniently, in the
picture on the left).  I REALLY liked the look of Left, but was stuck with
Jekyll's code formatting that was...less than ideal. Jekyll is pluggable (and
people have made plugins to fix this sort of thing), but I still
didn't have enough experience at that time
to be able to deal with the plugins in an intelligent manner.


## Octopress = Jekyll + Plugins + <3

During my plugin party, I discovered [Octopress][octopress], which
basically had EVERYTHING I wanted wrapped with a templated bow. I loved the way
it rendered code, it supported things like [fenced code blocks][gfm] , and it
seemed REALLY simple to update and deploy code.  The thing I DIDN'T like was
that NEARLY EVERY DAMN OCTOPRESS SITE LOOKS EXACTLY THE SAME! I know I said
that I'm not a web developer and didn't want to tweak styles a great bit, but
damn - couldn't I get a BIT of differentiation? That can be done, but you're
still editing styles (so some CSS is necessary - but not much). After
evaluating all three options, I opted for Octopress.

## Documentation? Who knew!?

Octopress.org has [GREAT documentation][oc setup] for
getting setup with an Octopress blog. They even have documentation for setting
up Ruby 1.9.3 on rbenv or RVM (I recommend rbenv for reasons beyond this blog),
so make sure to check it out if you're unfamiliar with either. To not reinvent
that documented wheel, make sure to check out that site to get Octopress setup
(I recommend cloning all repositories to somewhere in your home directory
like ~/src or ~/repos, but other than that their docs are solid). Normally
I post the dirty technical details, but the point of this post is to outline
WHY I made the decision I did and WHY it works for ME.


## Pretty code is pretty

I'm not gonna lie - syntactical highlighting with the [Solarized][solarized]
theme was pretty sexy. Let's look at some Ruby:

``` ruby
def create
    if resource[:value_type] == :boolean
      unless resource[:value].first.to_s =~ /(true|false)/i
        raise Puppet::Error, "Valid boolean values are 'true' or 'false', you specified '#{resource[:value].first}'"
      end
    end

    if File.file? resource[:path]
      plist = read_plist_file(resource[:path])
    else
      plist = OSX::NSMutableDictionary.alloc.init
    end

    case resource[:value_type]
    when :integer
      plist_value = Integer(resource[:value].first)
    when :boolean
      if resource[:value].to_s =~ /false/i
        plist_value = false
      else
        plist_value = true
      end
    when :hash
      plist_value = resource[:value].first
    else
      plist_value = resource[:value]
    end

    plist[resource[:key]] = plist_value

    write_plist_file(plist, resource[:path])
  end
```

How about some Puppet code?

{% codeblock Plist Management lang:puppet %}
property_list_key { 'simple':
  ensure => present,
  path => '/tmp/com.puppetlabs.puppet',
  key    => 'simple',
  value  => 'value',
}

property_list_key { 'boolean':
  ensure     => present,
  path       => '/tmp/com.puppetlabs.puppet',
  key        => 'boolean',
  value      => false,
  value_type => 'boolean',
}

property_list_key { 'hashtest':
  ensure     => present,
  path       => '/tmp/com.puppetlabs.puppet',
  key        => 'hashtest',
  value      => { 'key' => 'value' },
  value_type => 'hash'
}
{% endcodeblock %}

Doesn't that look awesome? What if you have a [gist][gist]? You can embed those too
(and they're also linkable):

{% gist 3218523 %}

Want to know how much code it took to do that?  About this much:

```
{% gist 3218523 %}
```

For a blog with a ton of code, something like this is pretty damn important.


## Markdown formatting

[Markdown][markdown] is becoming more prolific as a lightweight way to format
text. If you've used Github to comment on code, then you've probably used
markdown syntax at some point in your life. You'll notice the recurring goal of
being quick and precise, and markdown really 'does it for me'.


## Workflow++

It's really liberating to write a blog post in (insert favorite text editor).
Travelling as often as I do, I'm usually jotting down notes in vi because I'm
in a plane or somewhere without internet access. Octopress not only lets you
write offline, but it will let you generate and preview your site locally
while being offline. That's totally handy.  My workflow typically looks like
this:

* **Generate a post**

To start a new post, you can create a new file by hand, or use the
provided Rakefile to scaffold it out for you (NOTE: to see all commands
available to the Rakefile, you can run `rake -T`). Here's how to scaffold
with rake:

```
$ rake 'new_post["Title of my post"]'
$ vi source/_posts/2013-01-17-title-of-my-post.markdown
```

* **Edit the post file**
* **Generate the site content**

Remember that Octopress/Jekyll generates static content to be uploaded to
your host. With every change, you'll need to re-generate the content. Yeah,
that kinda sucks, but that action has been abstracted away in the Rakefile
with the following command:

```
$ rake generate
```

* **Display and view the site**
```
$ rake preview
```

The following command will serve up your page over Webrick in the terminal,
just navigate to <http://localhost:4000> in your browser to see a local copy of
how your site will look once deployed. Once you're done, just do a Control+c
from your terminal to cancel the process.

* **Edit/generate/preview until done**
* **Commit your code and deploy**

Because everything is a series of flat-files, you can use git to keep it all
under version control. Did you make a mistake? Revert to a previous commit.
Deploying to your blog is similarly easy, as you'll see in the next steps


## Be a unique snowflake

I [used this article](http://blog.bigdinosaur.org/changing-octopresss-header/) to help
change the color scheme of my blog, and [checked out a list of other Octopress sites](https://github.com/imathis/octopress/wiki/Octopress-Sites)
to steal a couple of other tweaks. That's all I needed for customization, but
if you need more knobs they're there for you to twist.

## Hosted by your pals at Github

[Github Pages][ghp] is a free service for any Github user (again free to join) and
is an EXCELLENT FIT for an Octopress blog. As you would expect, Octopress
has [great documentation for enabling deployment to Github Pages][deploy].
If you have your own custom domain, you can STILL use Github Pages
(hint, the instructions are on that page too).


## Fuck it. Ship it.

The act of updating a blog has GOT to be frictionless. Is this simple enough
for you:

```
rake deploy
```

Yep. That's all it takes to deploy code to your blog once you've setup Github
Pages. Forget about logging in, wading through a GUI, cutting/pasting content,
FTPing things up to a host, and any of that crap. I've done that. 'That' sucks.


## Write your own damn blog

You now have no excuse to NOT share all the cool things you're doing. Seriously,
if you can EDIT A TEXT FILE then you can 'maintain' a blog.


[markdown]: http://daringfireball.net/projects/markdown/
[pl]: http://www.puppetlabs.com
[gl]: http://glarizza.posterous.com
[gist]: http://gist.github.com
[jekyll]: https://github.com/mojombo/jekyll
[octopress]: http://octopress.org
[solarized]: http://ethanschoonover.com/solarized
[gfm]: http://github.github.com/github-flavored-markdown/)
[ghp]: http://pages.github.com/
[deploy]: http://octopress.org/docs/deploying/github/
[oc setup]: http://octopress.org/docs/setup/
