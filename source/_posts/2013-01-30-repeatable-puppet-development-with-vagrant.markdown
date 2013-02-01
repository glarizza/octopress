---
layout: post
title: "Repeatable Puppet development with Vagrant"
date: 2013-01-30 09:21
comments: true
categories: ['vagrant', 'puppet', 'devops', 'automation']
---

I miss testing code in production. In smaller organizations, 'testing' and
'development' can sometimes consist of making changes directly on a server,
walking to an active machine, and hoping things work. Once you were done, you
MIGHT document what changes you made, but more often than not you kept that
information in your head and referred to it later.

I lied - that is everything that sucks about manual configuration of machines.

The best way to get out of this rut is to get addicted to automating first the
menial tasks on your machines, and then work your way up from there. We STILL
have the problem, though, of doing this in production - that's what this post
is meant to address.

What we want is the ability to spin up a couple of test nodes for the purpose
of testing our automation workflow BEFORE it gets committed and applied to our
production nodes. This post details using [Vagrant][vagrant] and
[Puppet][puppet] to both establish a clean test environment and also test
automation changes BEFORE applying them to your production environment.

[Puppet][puppet] is a Configuration Management tool that automates all the
annoying aspects of manual configuration out of your infrastructure. The bulk
of its usage is beyond the scope of THIS post, however we're going to be using
it as the means to describe the changes we want to make on our systems.

[Vagrant][vagrant] is a magical project that uses minimal VM templates (boxes)
to spin up clean virtualized environments on your workstation for the purpose
of testing changes. Currently, it only supports a Virtualbox backend, but its
creator, [Mitchell Hashimoto][mh], has [teased a preview of upcoming VMware][preview]
integration that SHOULD be coming any day now. In this post, Vagrant will be
the means by which we spin up new VMs for development purposes

## Getting setup

The only moving piece you need installed on your system is [Vagrant][vagrant].
Fortunately, Mitchell [provides native package installers][vdl] on his website for
downloading Vagrant. If you've never used Vagrant before, and you AREN'T a Ruby
developer who maintains multiple Ruby versions on your system, then you'll want
to opt for the native package installer since it's the easiest method to get
Vagrant installed (and, on Macs, Vagrant embeds its own Ruby AND Rubygems
binaries in the Package bundle...which is kind of cool).

IF, however, you are developing in Ruby and you use [RVM][rvm] or [Rbenv][rbenv]
to maintain multiple copies of Ruby on your system, then you'll want to favor
installing Vagrant via Rubygems a la:

{% codeblock lang:bash %}
$ gem install vagrant --no-ri --no-rdoc
{% endcodeblock %}

If you have no idea how to use RVM or Rbenv - stick with the [native installers][vdl] :)

Puppet does NOT need to be on your workstation since we're only going to be using it on the VMs that
Vagrant spins up - so don't worry about Puppet yet.


## My kingdom for a box

Vagrant uses box files as templates from which to spin up a new virtual machine
for development purposes. [There are sites that host boxes available for download][vb],
OR, you could use [an awesome project called Veewee][veewee] to build your own.
Again, building your box file is outside the scope of this article, so just
make sure you download a box with an OS that's to your liking.  This box DOES
NOT need to have Puppet preinstalled - in fact, it's probably better that it
doesn't (because the version will probably be old, and we're going to work
around this anyways). I'm going to choose a CentOS 6.3 box that the SE team at
Puppet Labs uses for demos, but, again, it's up to you.


## Vagrantfile, assemble!

Now that we've got the pieces we need, let's start stitching together
a repeatable workflow. To do that, we'll need to create a directory for this
project and a `Vagrantfile` to direct Vagrant on how it should setup your VM.
I'm going to use `~/src/vagrant_projects` for the purpose of this demo:

{% codeblock lang:bash %}
$ mkdir -p ~/src/vagrant_projects
$ cd ~/src/vagrant_projects
$ vim Vagrantfile
{% endcodeblock %}

Let's take a look at a sample Vagrantfile that I use to get Puppet installed on
a box:

{% codeblock lang:ruby Vagrantfile %}
Vagrant::Config.run do |config|
  config.vm.box       = "centos-6.3-x86_64"
  config.vm.box_url   = "https://saleseng.s3.amazonaws.com/boxfiles/CentOS-6.3-x86_64-minimal.box"
  config.vm.host_name = "development.puppetlabs.vm"
  config.vm.network :hostonly, "192.168.33.10"
  config.vm.forward_port 80, 8084
  config.vm.provision :shell, :path => "centos_6_x.sh"
end
{% endcodeblock %}

Stepping through this file line-by-line, the first two `config.vm` lines
establish the box we want to use for our development VM as well as the URL to
the box file where it can be downloaded (in the event that it does not exist on
our system). Because, initially, this box will NOT be known to Vagrant, it will
attempt to reach out to that address and download it (note that the URL to THIS
PARTICULAR BOX is subject to change - please find a box file that works for you
and substitute its URL in the `config.vm.box_url` config setting).  The next
three lines define the machine's hostname, the network type, and the IP address
for this VM. In this case, I'm using a host-only network and giving it an IP
address on a made-up 192.168.33.0/24 subnet (feel free to use your own
private IP range as long as it doesn't conflict with anything). The next line
is forwarding port 80 on the VM to port 8084 on my local laptop - this allows
you to test out web services by simply navigating to http://localhost:8084
from your web browser.  I'll save explaining the last line for the next
section.

**NOTE**: For more documentation on these settings, [visit Vagrant's documentation site][vdocs]
as it's quite good


## Getting Puppet on your VM

The final line in the sample Vagrantfile runs what's called the 'Shell
Provisioner' for Vagrant. Essentially, it runs a shell script on the VM once
it's been booted and configured. What does this shell script do?

{% codeblock centos_6_x.sh lang:bash https://github.com/hashicorp/puppet-bootstrap/blob/master/centos_6_x.sh %}
#!/usr/bin/env bash
# This bootstraps Puppet on CentOS 6.x
# It has been tested on CentOS 6.3 64bit

set -e

REPO_URL="http://yum.puppetlabs.com/el/6/products/i386/puppetlabs-release-6-6.noarch.rpm"

if [ "$EUID" -ne "0" ]; then
  echo "This script must be run as root." >&2
  exit 1
fi

if which puppet > /dev/null 2>&1; then
  echo "Puppet is already installed"
  exit 0
fi

# Install puppet labs repo
echo "Configuring PuppetLabs repo..."
repo_path=$(mktemp)
wget --output-document=${repo_path} ${REPO_URL} 2>/dev/null
rpm -i ${repo_path} >/dev/null

# Install Puppet...
echo "Installing puppet"
yum install -y puppet > /dev/null

echo "Puppet installed!"
{% endcodeblock %}

As you can see, it sets up the Puppet Labs el6 repository containing the
current packages for Puppet/Facter/Hiera/PuppetDB/etc and installs the most
recent version of Puppet and Facter that are in the repository. This will
ensure that you have the most recent version of Puppet on your VM, and you
don't need to worry about creating a new box every time Puppet releases a new
version.

This code [came from Mitchell's puppet-bootstrap repo][shell] where he
maintains a list of scripts that will bootstrap Puppet onto many of the common
operating systems out there. This code was current as of the initial posting
date of this blog, but make sure to check that repo for any updates.  If you're
maintaining your OWN provisioning script, consider filing pull requests against
Mitchell's repo so we can ALL benefit from good code and don't have to keep
creating 'another wheel' just to provision Puppet on VMs!


## Spin up your VM

Once you've created a `Vagrantfile` in a directory, the next logical thing to
do is to test out Vagrant and fire up your VM. Let's first check the status of
the vm:

{% codeblock lang:bash %}
$ vagrant status

Current VM states:

default                  not created

The environment has not yet been created. Run `vagrant up` to
create the environment.
{% endcodeblock %}

As expected, this VM has yet to be created, so let's do that by doing a `vagrant up`

{% codeblock %}
$ vagrant up

[default] Box centos-6.3-x86_64 was not found. Fetching box from specified
URL...
[vagrant] Downloading with Vagrant::Downloaders::HTTP...
[vagrant] Downloading box:
https://saleseng.s3.amazonaws.com/boxfiles/CentOS-6.3-x86_64-minimal.box
[vagrant] Extracting box...
[vagrant] Verifying box...
[vagrant] Cleaning up downloaded box...
[default] Importing base box 'centos-6.3-x86_64'...
[default] The guest additions on this VM do not match the install version of
VirtualBox! This may cause things such as forwarded ports, shared
folders, and more to not work properly. If any of those things fail on
this machine, please update the guest additions and repackage the
box.

Guest Additions Version: 4.1.18
VirtualBox Version: 4.1.23
[default] Matching MAC address for NAT networking...
[default] Clearing any previously set forwarded ports...
[default] Forwarding ports...
[default] -- 22 => 2222 (adapter 1)
[default] -- 80 => 8084 (adapter 1)
[default] Creating shared folders metadata...
[default] Clearing any previously set network interfaces...
[default] Preparing network interfaces based on configuration...
[default] Booting VM...
[default] Waiting for VM to boot. This can take a few minutes.
[default] VM booted and ready for use!
[default] Configuring and enabling network interfaces...
[default] Setting host name...
[default] Mounting shared folders...
[default] -- v-root: /vagrant
[default] Running provisioner: Vagrant::Provisioners::Shell...
Configuring PuppetLabs repo...
warning: 
/tmp/tmp.FvW0K7FJWU: Header V4 RSA/SHA1 Signature, key ID 4bd6ec30: NOKEY
Installing puppet
warning: 
rpmts_HdrFromFdno: Header V4 RSA/SHA1 Signature, key ID 4bd6ec30: NOKEY
Importing GPG key 0x4BD6EC30:
 Userid : Puppet Labs Release Key (Puppet Labs Release Key) <info@puppetlabs.com>
 Package: puppetlabs-release-6-6.noarch (installed)
 From   : /etc/pki/rpm-gpg/RPM-GPG-KEY-puppetlabs
Warning: RPMDB altered outside of yum.
Puppet installed!
{% endcodeblock %}

Vagrant first noticed that we did not have the CentOS box on our machine, so it
downloaded, extracted, and verified the box before importing it and creating
our custom VM. Next, it configured the VM's network settings according to our
Vagrantfile, and finally it provisioned the box using the script we passed in
the Vagrantfile.

We've now got a VM running and Puppet is installed. Let's ssh to our VM
and check the Puppet Version:

{% codeblock lang:bash %}
$ vagrant ssh

Last login: Tue Jul 10 22:56:01 2012 from 10.0.2.2
[vagrant@development ~]$ puppet --version
3.0.2
[vagrant@development ~]$ hostname
development.puppetlabs.vm
[vagrant@development ~]$ exit
logout
Connection to 127.0.0.1 closed.

$ vagrant destroy -f
[default] Forcing shutdown of VM...
[default] Destroying VM and associated drives...
{% endcodeblock %}

Cool - so we demonstrated that we could ssh into the VM, check the Puppet
version, check the hostname to ensure that Vagrant had set it correctly, exit
out, and then we finally destroyed the VM with `vagrant destroy -f`.  The next
step is to actually configure Puppet to DO something with this VM...

## Using Puppet to setup your node

The act of GETTING a clean VM is all well and good (and is probably magic
enough for most people out there), but the purpose of this post is to
demonstrate a workflow for testing out Puppet code changes. In the previous
step we showed how to get Puppet installed, but we've yet to demonstrate how
to use Vagrant's built-in [Puppet provisioner][pp] to configure your VM. Let's use
the example of a developer wanting to spin up a LAMP stack. To manually
configure that would require installing a number of packages, editing a number
of config files, and then making sure services were installed (among other
things). We're going to use some of the Puppet modules from the [Puppet
Forge][forge] to tackle these tasks and make Vagrant automatically configure
our VM.

## Scaffolding Puppet

We need a way to pass our Puppet code to the VM Vagrant creates. Fortunately,
Vagrant has a way to define [Shared Folders][vsf] that can be shared from your
workstation and mounted on your VM at a particular mount point. Let's modify
our Vagrantfile to account for this shared folder:


{% codeblock lang:ruby Vagrantfile %}
Vagrant::Config.run do |config|
  config.vm.box       = "centos-6.3-x86_64"
  config.vm.box_url   = "https://saleseng.s3.amazonaws.com/boxfiles/CentOS-6.3-x86_64-minimal.box"
  config.vm.host_name = "development.puppetlabs.vm"
  config.vm.network :hostonly, "192.168.33.10"
  config.vm.forward_port 80, 8084
  config.vm.provision :shell, :path => "centos_6_x.sh"

  # Puppet Shared Folder
  config.vm.share_folder "puppet_mount", "/puppet", "puppet"
end
{% endcodeblock %}

The syntax for the `config.vm.share_folder` line is that the first argument is
a logical name for the shared folder mapping, the second argument is the path
IN THE VM where this folder will be mounted (so, a folder called 'puppet' in
the root of the filesystem), and the last argument is the path to the folder ON
YOUR WORKSTATION that will be mounted in the VM (it can be a full or relative
path - which is what we've done here). This folder hasn't been created yet, so
let's create it (and a couple of subfolders):

{% codeblock lang:bash %}
$ cd ~/src/vagrant_projects
$ mkdir -p puppet/{manifests,modules}
{% endcodeblock %}

This command will create the `puppet` directory in the same directory that
contains our Vagrantfile, and then two subdirectories, `manifests` and
`modules`, that will be used by the Puppet provisioner later. Now that we've
told Vagrant to create our shared folder, and we've created the folder
structure, let's bring up the VM with `vagrant up` again, ssh into the VM with
`vagrant ssh`, and then check to see that the folder has been mounted.

{% codeblock lang:bash %}
$ vagrant up

<output suppressed - see above for example output>

$ vagrant ssh

Last login: Tue Jul 10 22:56:01 2012 from 10.0.2.2
[vagrant@development ~]$ ls /puppet
manifests  modules
{% endcodeblock %}

Great! We've setup a shared folder. To further test it out, you can try
dropping a file in the puppet directory or one of its subdirectories - it
should immediately show up on the VM without having to recreate the VM (because
it's a shared folder). There are pros and cons with this workflow - the main
pro is that changes you make on your workstation will immediately be reflected
in the VM, and the main con is that you can't symlink folders INSIDE
the shared folder on your workstation because of the nature of symlinks.


## Installing the necessary Puppet Modules

Since we've already spun up a new VM and ssh'd into it, let's use our VM to
download modules we're going to need to setup our LAMP stack:

{% codeblock lang:bash %}
[vagrant@development ~]$ puppet module install puppetlabs/apache --target-dir /puppet/modules/
Notice: Preparing to install into /puppet/modules ...
Notice: Downloading from https://forge.puppetlabs.com ...
Notice: Installing -- do not interrupt ...
/puppet/modules
└─┬ puppetlabs-apache (v0.5.0-rc1)
  ├── puppetlabs-firewall (v0.0.4)
  └── puppetlabs-stdlib (v3.2.0)

[vagrant@development ~]$ puppet module install puppetlabs/mysql --target-dir /puppet/modules/
Notice: Preparing to install into /puppet/modules ...
Notice: Downloading from https://forge.puppetlabs.com ...
Notice: Installing -- do not interrupt ...
/puppet/modules
└── puppetlabs-mysql (v0.6.1)

[vagrant@development ~]$ ls /puppet/modules/
apache  concat  firewall  mysql  stdlib
{% endcodeblock %}

The `puppet` binary has a module subcommand that will connect to the [Puppet Forge][forge]
to download Puppet modules and their dependencies. The commands we used will
install Puppet Labs' `apache` and `mysql` modules (and their dependencies).
We're also passing the `--target-dir` argument that will tell the `puppet
module` subcommand to install the module into our shared directory (instead of
Puppet's default module path).

I'm choosing to use `puppet module` to install these modules, but there are
a multitude of other methods you can use (from downloading the modules directly
out of [Github][gh] to using a tool like [librarian-puppet][lp]). The point is
that we need to ultimately get the modules into the `modules` directory
in our shared `puppet` folder - however you want to do that works for me :)

Once the modules are in `puppet/modules`, we're good. You only ever need to do
this step ONCE. Because this folder is a shared folder, you can now `vagrant
up` and `vagrant destroy` to your heart's content - Vagrant will not remove the
content in our shared folder when a VM is destroyed. Remember, too, that any
changes made to those modules from either the VM or on your Workstation will be
IMMEDIATELY available to both.

Since we're now done with the VM for now, let's destroy it with `vagrant destroy`

{% codeblock %}
$ vagrant destroy
{% endcodeblock %}


## Classifying your development VM

The modules we installed are a framework that we will use to configure the
node. The act of directing the actions that Puppet should take on a particular
node is called ['Classification'][classify]. Puppet uses a file called [site\.pp][site]
to map Puppet code with the corresponding 'node' (or, in our case, our VM) that
should receive it. Let's create a site.pp file and open it for editing:

{% codeblock %}
$ cd ~/src/vagrant_projects
$ vim puppet/manifests/site.pp
{% endcodeblock %}

Let's create a site.pp that will setup the LAMP stack on our
`development.puppetlabs.vm` that we create with Vagrant:

{% codeblock lang:puppet ~/src/vagrant_projects/manifests/site.pp %}
node 'development.puppetlabs.vm' {
  # Configure mysql
  class { 'mysql::server':
    config_hash => { 'root_password' => '8ZcJZFHsvo7fINZcAvi0' }
  }
  include mysql::php

  # Configure apache
  include apache
  include apache::mod::php
  apache::vhost { $::fqdn:
    port    => '80',
    docroot => '/var/www/test',
    require => File['/var/www/test'],
  }

  # Configure Docroot and index.html
  file { ['/var/www', '/var/www/test']:
    ensure => directory
  }

  file { '/var/www/test/index.php':
    ensure  => file,
    content => '<?php echo \'<p>Hello World</p>\'; ?> ',
  }

  # Realize the Firewall Rule
  Firewall <||>
}
{% endcodeblock %}

Again, the point of this post is not about writing Puppet code but more about
testing the Puppet code you write. The above node declaration will setup MySQL
with a root password of 'puppet', setup Apache and a VHost for
`development.puppetlabs.vm` with a docroot out of `/var/www/test`, setup an
`index.php` file for Apache, and setup a Firewall rule to allow access through
to port 80 on our VM.


## Setting up the Puppet provisioner for Vagrant

We're going to have to modify our `Vagrantfile` one more time to tell Vagrant
to use the [Puppet provisioner][pp] to execute our Puppet code and setup our VM:

{% codeblock lang:ruby Vagrantfile%}
Vagrant::Config.run do |config|
  config.vm.box       = "centos-6.3-x86_64"
  config.vm.box_url   = "https://saleseng.s3.amazonaws.com/boxfiles/CentOS-6.3-x86_64-minimal.box"
  config.vm.host_name = "development.puppetlabs.vm"
  config.vm.network :hostonly, "192.168.33.10"
  config.vm.forward_port 80, 8084
  config.vm.provision :shell, :path => "centos_6_x.sh"

  # Puppet Shared Folder
  config.vm.share_folder "puppet_mount", "/puppet", "puppet"

  # Puppet Provisioner setup
  config.vm.provision :puppet do |puppet|
    puppet.manifests_path = "puppet/manifests"
    puppet.module_path    = "puppet/modules"
    puppet.manifest_file  = "site.pp"
  end
end
{% endcodeblock %}

Notice the block for the Puppet provisioner that sets up the manifest path (i.e. where
to find `site.pp`), the module path (i.e. where to find our Puppet modules),
and the name of our manifest file (i.e. `site.pp`). Again, [this is all documented][pp]
on the [Vagrant][vagrant] documentation page should you need to use it for
reference.

This bumps the number of provisioners in our `Vagrantfile` to two, but which
one goes first? Vagrant will iterate through the `Vagrantfile` procedurally, so
the Shell provisioner will always get checked first and then the Puppet
provisioner will get checked second.  This allows us to be certain that Puppet
will always be installed before attempting to use the Puppet provisioner. You
could continue to add as many provisioning blocks as you like - Vagrant will
iterate through them procedurally as it encounters them.


## Give the entire workflow a try

Now that we have our Vagrantfile finalized, our Puppet directory structure
setup, our Puppet modules installed, and our `site.pp` file set to classify our
new VM, let's actually let Vagrant do what it does best and setup our VM:

{% codeblock %}
$ vagrant up
{% endcodeblock %}

You should see Vagrant use the Shell provisioner to install Puppet, hand off to
the Puppet provisioner, and then use Puppet to setup a LAMP stack on our VM.
After everything completes, try visiting [http://localhost:8084](http://localhost:8084)
in your web browser and see if you get a shiny "Hello World" staring back at
you.  If you do - Awesome!  If you don't, check the error messages to determine
if there are typos in the Puppet code or if something went wrong in the `Vagrantfile`.


## Where do you take it from here?

The first thing to do is to take the Vagrantfile you've created and put it
under revision control so you can track the changes you make. I personally have
a couple of workflows up on [Github][gh] that I use as templates when I'm
testing out something new. You'll probably find that your Vagrantfile won't
change much - just the modules you use for testing.

Now that you understand the pattern, you can expand it to fit your workflow.
Single-vm projects are great when you're testing a specific component, but
the next logical step is to test out multi-tiered components/applications.  In these instances,
[Vagrant has the ability to spin up multiple VMs from a single Vagrantfile][multivm].
That workflow saves a TON of time and lets you create your own private network
of VMs for the purpose of simulating changes.  That's a post for another time,
though...


## Get involved

Stay tuned to the [Vagrant website][vagrant] for updates on the VMware
provisioner.  Stability with Virtualbox has notoriously been an issue, but, as
of this posting, things have been relatively rock-solid for me (using
Virtualbox version 4.1.23 on OS X).

If you want to keep up-to-date on all things Vagrant, follow [Mitchell][mh] on
Twitter, check out `#vagrant` on Freenode, join [the Vagrant list][vu], and
check out Google for what other folks have done!

A GIANT thank you to [Mitchell Hashimoto][mh] for all the work he's done on
Vagrant - I can't count the number of hours it's saved me personally (let ALONE
everyone at [Puppet Labs][puppet]!

[puppet]: http://www.puppetlabs.com
[vagrant]: http://www.vagrantup.com
[mh]: http://twitter.com/mitchellh
[preview]: http://vimeo.com/58059557
[vdl]: http://downloads.vagrantup.com
[rvm]: https://rvm.io/
[rbenv]: https://github.com/sstephenson/rbenv/
[vb]: http://vagrantbox.es
[veewee]: https://github.com/jedi4ever/veewee
[vdocs]: http://docs.vagrantup.com/v1/docs/vagrantfile.html
[shell]: https://github.com/hashicorp/puppet-bootstrap/blob/master/centos_6_x.sh
[forge]: http://forge.puppetlabs.com
[vsf]: http://docs.vagrantup.com/v1/docs/config/vm/share_folder.html
[gh]: http://github.com
[lp]: https://github.com/rodjek/librarian-puppet
[classify]: http://docs.puppetlabs.com/puppet/3/reference/lang_summary.html#resources-classes-and-nodes
[site]: http://docs.puppetlabs.com/learning/agent_master_basic.html#sitepp
[pp]: http://docs.vagrantup.com/v1/docs/provisioners/puppet.html
[multivm]: http://docs.vagrantup.com/v1/docs/multivm.html
[vu]: http://groups.google.com/group/vagrant-up
[pull]: https://github.com/hashicorp/puppet-bootstrap/pull/7
