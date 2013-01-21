---
layout: post
title: "Using Veewee to build OS X VMs"
date: 2013-01-20 19:36
comments: true
categories: ['veewee', 'os x', 'fusion', 'vm']
---

I hate [Tim Sutton][ts] in the same way I hate [Steven Singer][ss]. I hate Tim
because he actually IMPLEMENTS some of the things on my list of 'shit to tinker
with when you get some free time' instead of just TALKING about them. He and
[Pepijn Bruienne][pb] have been working on some code for [Veewee][veewee] that
will allow you to automate the creation of OS X VMs in VMware Fusion. For those
who are interested, you can [check out the pull request containing this code][vpull]
and comment/help them out. For everyone else, read on and entertain the idea...


## Prerequisites:

1. OS X
1. VMware Fusion
2. Git
3. Ruby 1.9.3 and rbenv
4. Mountain Lion (10.8) installer application

#### OS X

This walkthrough assumes that you're running OS X on your development
workstation. That's what I use, and it's the workflow I know.

#### VMware Fusion

If you've used [Veewee][veewee] before, odds are good that you MIGHT have used
it to build baseboxes for [Vagrant][vagrant] and/or Virtualbox. Veewee
+ Vagrant is a post for ANOTHER time, but in short it's an awesome workflow for
testing automation like [Puppet](http://www.puppetlabs.com) on Linux or
Windows. When I originally tried using Veewee to build OS X VMs, I had
absentmindedly tried to do this using Virtualbox...which isn't supported. As
such, VMware Fusion is the only virtualization platform (as of this posting
date) that is supported with this method. I'm using VMware Fusion 5 pro, but
YMMV.

#### Git

Because the code that supports OS X only exists in Tim's fork of Veewee (as of
this posting date), we're going to have to install/use Veewee from source. That
introduces a bit more complexity, but hold my hand - we'll get through this
together. Git comes with the XCode Command Line Tools or can be [installed with native packages][gitdl].

#### Ruby 1.9.3 and rbenv

Veewee uses the `gssapi` gem which requires Ruby 1.9.1 or higher. The problem,
though, is that the version of Ruby that comes with OS X is 1.8.7. There are
typically two camps when it comes to getting a development version of Ruby on
OS X: [rbenv][rbenv] or [RVM][rvm]. I recommend [rbenv][rbenv] because it
doesn't screw with your path and it's a bit more lightweight, so that's the
path I'm going to take in this writeup. [Octopress has instructions][rbenv]
for getting [rbenv][rbenv] on your machine - so make sure to check those out
for this step. The instructions describe using rbenv to install Ruby version
1.9.3 - and that's the version we'll use here.

#### Mountain Lion installer application

This workflow supports creating a VM for Mountain Lion (10.8), but it SHOULD
also work for Lion (10.7). The final piece of our whole puzzle is the installer
application from the App Store. Get that application somehow and drop it in the
`/Applications` directory (that's where the App store puts it by default). We
REALLY only need a single disk image from the installer, and we'll get that
next.


## Some assembly required...

Now that you have all the pieces, let's tie this FrankenVM up with some code
and ugly bash, shall we? Splendid.

#### Copy out the installation disk image

In the last step, we downloaded the `Install OS X Mountain Lion.app` file from
the App Store to `/Applications`, but we'll want to extract the main
installation disk image somewhere where we can work with it. I'm going to make
a copy on the Desktop so we don't screw up the main installer:

{% codeblock lang:bash %}
$ cp /Applications/Install\ OS\ X\ Mountain\ Lion.app/Contents/SharedSupport/InstallESD.dmg ~/Desktop
{% endcodeblock %}

Beautiful. This should take a minute as it's a sizeable file, but in the end
you'll have the installation disk image on your Desktop. For now this is fine,
but we'll be revisiting it later...

#### Clone Tim's fork of Veewee

As if the writing of this blog post, the code you need is ONLY in Tim's fork,
so let's pull that down to somewhere where we can work with it:

{% codeblock lang:bash %}
## I prefer to work with code out of a 'src' directory in my home directory
$ mkdir ~/src
$ cd ~/src

$ git clone http://github.com/timsutton/veewee
$ cd ~/src/veewee
{% endcodeblock %}

#### Install Gems for veewee

We now have the Veewee source in `~/src/veewee`, but we need to ensure all the
Rubygems necessary to make Veewee work have been installed. We're going to do
this with [Bundler][bundler]. Let's switch to Ruby 1.9.3 and get Bundler
installed:

{% codeblock lang:bash %}
$ cd ~/src/veewee
$ rbenv local 1.9.3
$ gem install bundler
{% endcodeblock %}

Next, let's use [bundler][bundler] to install the rest of the gems we need to
use Veewee:

{% codeblock lang:bash %}
$ bundle install
{% endcodeblock %}

Once that command completes, Bundler will have installed all the necessary gems
for Veewee and we can move on.

#### Define your new VM

Veewee has templates for most operatingsystems that can be used to spin up
a 'vanilla' VM. Tim's code provides a template called 'OSX-10.8.2' containing
the necessary scaffolding for building a vanilla 10.8.2 VM. Let's create a new
VM project based on this template called 'osx-vm' with the following:

{% codeblock lang:bash %}
$ cd ~/src/veewee
$ bundle exec veewee fusion define 'osx-vm' 'OSX-10.8.2'
{% endcodeblock %}

This will create `definitions/osx-vm` inside the Veewee directory with the
template code from `templates/OSX-10.8.2`. We're almost ready to let Veewee
create our VM, but we need an installation 'ISO' first...

#### Prepare an 'ISO' for OS X

The `prepare_veewee_iso.sh` script in Veewee's `templates/OSX-10.8.2/prepare_veewee_iso`
directory provides awesome detail as to why we can't use the vanilla InstallESD.dmg
file to install 10.8 in our new VM. Feel free to open that file and read
through the detail, or [check out the details online][isoreadme] for more
information. Let's use that script and prepare an installation 'ISO' for our
new VM:

{% codeblock lang:bash %}
$ cd ~/src/veewee
$ mkdir iso
$ sudo templates/OSX-10.8.2/prepare_veewee_iso/prepare_veewee_iso.sh ~/Desktop/InstallESD.dmg iso/
{% endcodeblock %}

You'll need to be root to do this, but the script should handle everything
necessary to prepare the ISO and drop it into the `iso` directory we created.

#### Define any additional post-installation tasks

Veewee supports post-installation tasks through the `postinstall.sh` script in
the `definitions/osx-vm` folder. By default, this script will install
VMware tools, setup Vagrant keys, download the XCode Command Line Tools, and
install Puppet via Rubygems. Because this is all outlined in the `postinstall.sh`
script, you're free to modify this code or add your own steps. Here's the
current `postinstall.sh` script as of this posting:

{% codeblock lang:bash %}
date > /etc/vagrant_box_build_time
OSX_VERS=$(sw_vers -productVersion | awk -F "." '{print $2}')
# Install VMware tools if we were built with VMware
if [ -e .vmfusion_version ]; then
  TMPMOUNT=`/usr/bin/mktemp -d /tmp/vmware-tools.XXXX`
  hdiutil attach darwin.iso -mountpoint "$TMPMOUNT"
  installer -pkg "$TMPMOUNT/Install VMware Tools.app/Contents/Resources/VMware Tools.pkg" -target /
  # This usually fails
  hdiutil detach "$TMPMOUNT"
  rm -rf "$TMPMOUNT"
fi

# Installing vagrant keys
mkdir /Users/vagrant/.ssh
chmod 700 /Users/vagrant/.ssh
curl -k 'https://raw.github.com/mitchellh/vagrant/master/keys/vagrant.pub' > /Users/vagrant/.ssh/authorized_keys
chmod 600 /Users/vagrant/.ssh/authorized_keys
chown -R vagrant /Users/vagrant/.ssh

# Get Xcode CLI tools for Lion (at least to build Chef)
# https://devimages.apple.com.edgekey.net/downloads/xcode/simulators/index-3905972D-B609-49CE-8D06-51ADC78E07BC.dvtdownloadableindex
TOOLS=clitools.dmg
if [ "$OSX_VERS" -eq 7 ]; then
  DMGURL=http://devimages.apple.com/downloads/xcode/command_line_tools_for_xcode_os_x_lion_november_2012.dmg
elif [ "$OSX_VERS" -eq 8 ]; then
  DMGURL=http://devimages.apple.com/downloads/xcode/command_line_tools_for_xcode_os_x_mountain_lion_november_2012.dmg
fi
curl "$DMGURL" -o "$TOOLS"
TMPMOUNT=`/usr/bin/mktemp -d /tmp/clitools.XXXX`
hdiutil attach "$TOOLS" -mountpoint "$TMPMOUNT"
installer -pkg "$(find $TMPMOUNT -name '*.mpkg')" -target /
hdiutil detach "$TMPMOUNT"
rm -rf "$TMPMOUNT"
rm "$TOOLS"

# Get gems - we should really be installing rvm instead, since we can't even compile Chef or have a Ruby dev environment..
gem update --system
gem install puppet --no-ri --no-rdoc
# gem install chef --no-ri --no-rdoc
exit
{% endcodeblock %}


#### Build the VM

Everything we've done has all been for this step. With the ISO built, the VM
defined, and all necessary Gems installed, we can finally BUILD the vm with the
following command:

{% codeblock lang:bash %}
$ cd ~/src/veewee
$ bundle exec veewee fusion build osx-vm
{% endcodeblock %}

This process takes the longest - you should see VMware Fusion fire up, a new VM
get created, and follow the process of OS X being installed into the VM. When
it completes, your VM will have been created. Just like most Vagrant workflows,
the resultant vm will have a `vagrant` user whose password is also `vagrant`.
Feel free to login and ensure that everything looks good.


## Now what?

At this point, I would snapshot the VM before making any changes. Because
Virtualbox isn't yet supported with Veewee for building OS X VMs (and Vagrant
doesn't currently include VMware Fusion support for its workflow), this VM
isn't going to fit into a Vagrant workflow...yet. What you have, though, is
a vanilla OS X VM that can be built on-demand (or reverted to a snapshot) to
test whatever configuration changes you need to make (all 11.36 Gb worth of
it).

As you would expect, this is all pretty experimental for the moment. If you're
a Ruby developer who needs an OS X VM for testing purposes but have never
managed OS X and it's 'quirky' imaging process, this workflow is for you. For
everyone else, it's probably an academic proof-of-concept that's more
interesting from the point of view of "Look what you can do" versus "Let's make
this my primary testing workflow."

Credit goes to [Patrick Dubois][pd] for creating and managing the Veewee
project, and to [Tim Sutton][ts] and [Pepijn Bruienne][pb] - like I mentioned
before - for the work they've done on this. You can speak with them directly in
the ##osx-server chatroom on Freenode, or by checking them out on Twitter.

[ts]: http://github.com/timsutton
[ss]: http://ihatestevensinger.com/
[pb]: https://twitter.com/bruienne
[pd]: https://twitter.com/patrickdebois
[veewee]: http://github.com/jedi4ever/veewee
[vpull]: https://github.com/jedi4ever/veewee/issues/481
[vagrant]: http://www.vagrantup.com/
[gitdl]: http://git-scm.com/download/mac
[bundler]: http://gembundler.com/
[rbenv]: http://octopress.org/docs/setup/rbenv/
[rvm]: http://octopress.org/docs/setup/rvm/
[isoreadme]: https://github.com/timsutton/veewee/blob/feature/osx-guest/templates/OSX-10.8.2/prepare_veewee_iso/prepare_veewee_iso.sh
