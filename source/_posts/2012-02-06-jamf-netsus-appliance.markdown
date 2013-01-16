---
layout: post
title: "JAMF NetSUS Appliance"
date: 2012-02-06 11:22
comments: true
categories: 
---

## JAMF's NetSUS Appliance - Netboot in a Box

[Today, JAMF released a new appliance VM](https://jamfnation.jamfsoftware.com/viewProduct.html?id=180) based on Ubuntu 10.04 LTS (Lucid) that, for once, provides an 'out of the box' implementation of Netboot and Software Update Service WITHOUT requiring OS X hardware (based on [Reposado](http://github.com/wdas/reposado)) and [other open-source technologies.](http://twitpic.com/8glygw)

For those people who are relatively new to Linux, I thought I'd provide an easy walkthrough of how to get started using this VM on your Laptop.  I'm going to demo this using VMware Fusion instead of VirtualBox mainly because I prefer how Fusion handles Networking.  I've used many VMs, and Fusion always tends to be much more solid and stable for me.  If you don't have Fusion or don't WANT to use Fusion, feel free to use VirtualBox.

##Download and Convert the OVA File

[The appliance can be downloaded directly from JAMF](https://jamfnation.jamfsoftware.com/viewProduct.html?id=180) and comes as a VirtualBox OVA file.  [Rich Trouton has provided a great walkthrough](http://derflounder.wordpress.com/2012/02/06/converting-jamfs-netbootsus-appliance-to-vmware/) on converting the OVA file to a VMX file, so I'll link you to that post (since he did it so well).

##Start up the VM

Once you have your VMX file, you can open it up with Fusion and import it into your Virtual Machine Library.  There is one change we'll want to make, so click on the VM and open its Settings panel (this is different from Fusion 3.0 and 4.0 - I'll be describing the 4.0 method).  In the Settings panel, you'll want to select the Network Adapter icon and make sure the radio button next to "Share the Mac's network connection (NAT)" is selected.  This will create a local IP address so we can contact it directly from our Laptop.  Once you've done that, start the VM and get to the login screen (you'll need to click the Ok button through the message about logging into the NetSUS instance).  Feel free to login with the username 'shelluser' and the password 'shelluser'.  You should see a Message of the Day splash with one of the sections being "IP address for eth0:".  Note the IP address, we'll need it in the next section (if you don't see this, just run `ifconfig` from the command line and look for the IP address for the eth0 adapter - it should be in the format of 192.168.x.x).


##Prepare to Break Free!

It really sucks to work with VMs from WITHIN the VM window for various reasons (lack of Copy/Paste, loss of mouse, etc…), so we're going to setup a host entry on your local laptop so we can SSH into the VM as we need.  I'm going to give my VM a hostname of 'netsus.puppetlabs.vm' but you can name it whatever you want.  Let's first edit the /etc/hosts file on your laptop (NOT within the VM, this is ON your laptop) by running the following from the command line:

```
    sudo vim /etc/hosts
```

Feel free to use pico/nano to edit /etc/hosts in case you don't have any experience with vim.  We want to add the following line in the /etc/hosts file:

```
    192.168.217.154 netsus.puppetlabs.vm
```

**NOTE** SUBSTITUTE 192.168.217.154 WITH THE IP ADDRESS YOU GOT FROM THE PREVIOUS STEP!  That's the IP address that was assigned on MY laptop, and it will definitely be different from the IP address you got on YOUR computer.  Also, you don't HAVE to use 'netsus.puppetlabs.vm' - you can assign it any name you want.  Save and close /etc/hosts and then let's test connectivity from the command line by doing a ping:

```
    ping 192.168.217.154
```

As long as we have connection, we're fine (remember, substitute your IP Address for mine).  If you DON'T have connectivity, make sure your VM is running and review your /etc/hosts file to make sure the changes were saved (also, check the IP address on your VM again).


##Snapshot Time

The beauty of Fusion is that you can take snapshots and revert to them should you mess things up.  Let's take a snapshot now in case we screw things up in the following steps.  I'm using VMware Fusion 4.0, so I take a snapshot by clicking on the VM window, clicking on the Virtual Machine menu at the top, and then down to Snapshots.  From there, you can click the 'Take Snapshot' button and let it do its thing.  When it's done, hit the close button and return to the VM


##Enable SSH and Login

By default, the VM doesn't have the SSHD process running, so we can't yet SSH in from our laptop.  Enable this by doing the following from within the VM:

```
    sudo apt-get install openssh-server
```

Hit "Y" when it asks if it should download and install.  When it completes, switch back to your LAPTOP (i.e. NOT inside the VM), open up Terminal, and do the following:

```
    ssh shelluser@netsus.puppetlabs.vm
```

Type yes to add the SSH key to your known\_hosts file and use the password 'shelluser' to login.  Tada! Now you can work directly from Terminal and enable copy/pasting directly from your Terminal window.  You can always type `exit` to close the ssh connection when you're done.


##Install Puppet

So, it wouldn't be one of my write-ups if I DIDN'T get Puppet installed.  Actually, if you've only managed Macs, then you might not have seen the benefit of Puppet before.  Now that we're on Linux, you'll definitely see the benefit of Puppet (especially if you're not familiar with the service/package manipulation commands).  Puppet can help abstract that for you, so we're going to enable it.

The version of Puppet that's in the main package repository is quite old (0.25 as opposed to the current 2.7.10 version we support).  Let's add Puppet Labs' Apt Repository to the list of repositories queried so we can get a recent version of Puppet.  Do that by opening up the /etc/apt/sources.list file for editing with the following:

```
    sudo vim /etc/apt/sources.list
```

If it prompts you for your password, go ahead and enter it.  Also, if you're not familiar with vi or vim, I'll try and help you with the basics.  Arrow down to the line JUST above the line that begins with "deb http://" and press the i button on your keyboard to insert the following lines:

```
    deb http://apt.puppetlabs.com/ubuntu lucid main
    deb-src http://apt.puppetlabs.com/ubuntu lucid main
```

When you're done, hit the escape key and press ZZ (hold shift and hit Z twice, you want two capital-Z's) to save and close the file.  This will add Puppet Labs' repository to the list of repos queried.

Next, we'll need to add Puppet Labs' GPG key to the system so we can validate the packages that are downloaded.  From the terminal, execute the following:

```
     gpg --recv-key 4BD6EC30
```

The first time you run it, it will bother you about needing to create the keyserver file and won't run correctly.  Let's execute it correctly to receive the key:

```
     gpg --recv-key 4BD6EC30
```

Great, we should have the key, now we want to load it.  Do that with this command:

```
    gpg -a --export 4BD6EC30 | sudo apt-key add -
```

Finally, let's update apt so that it knows about the repository.  Do that with this command:

```
    sudo apt-get update
```

When it finishes running, you can finally install Puppet with the following command:

```
    sudo apt-get install puppet
```

Type "Y" when it asks you if it should download and install all the dependencies.  That's it!  Puppet should be installed!  Run the following to see what version we've installed:

```
    puppet --version
```

As of this writing, 2.7.10 is the current version.  As long as you don't have version 0.25 you should be good!


##Snapshots away

It would be wise to take another snapshot at this point.  Don't worry, I'll wait.

##Inspect the System

Puppet lets us see what's been installed on the VM.  Take a look at all the packages on the system by doing the following:

```
    sudo puppet resource package
```

Type your password when prompted and Puppet will return you with a list of all packages on the system and the versions that are running.  Similarly, you can see what services are running with the following:

```
    sudo puppet resource service
```

Notice that sshd is running and other services like netatalk (that enables AFP on Linux) should also be running.  Sweet.  We'll come back to Puppet later.


##Login to the Web GUI

If your VM is running, you should be able to open a web browser on your Laptop and navigate to the following address:

```
    https://netsus.puppetlabs.vm
```

Note the httpS as we'll need to use SSL to get into the Web GUI.  When it asks if you want to accept the self-signed cert, make sure to choose Yes.  You can login with the user 'webadmin' and the password 'webadmin'

Once you're logged in, [feel free to read through JAMF's instructions for enabling their services.](https://s3.amazonaws.com/jamfsoftware-content/downloads/NetBootSUS+Appliance_v1.0.pdf)  They do a much better job than I about walking you through that.


##Access to the Raw Files

Back in the Terminal window, you can access all the PHP/Shell magic that comprises the Web GUI in the /var/www/webadmin directory.  You can poke around through all the files (and use sudo to open some of the key bits) at your leisure.

Note that /etc/dhcpd.conf is the main configuration file for the DHCPD service and it's got some special bits to accommodate Apple Hardware.  

Also, /var/appliance has some magic (and the /var/appliance/configurefornetboot script has some sed statements for fixing a stock dhcpd.conf file and enabling those Apple Hardware bits).


##More Later?

That's it for now - I just wanted to get something online to show people how to setup and poke around in the VM.  Later I can show you how to manipulate and manage the VM with Puppet, which is pretty fun.  Your VM should get you started and let you play around locally, but if you want to test out Netboot and the like then you'll need an externally-accessible IP address (as apposed to the 192.168.x.x IP, which is only accessible from your Laptop).  You can change the settings in VMware Fusion to enable that (like we did in step 2).

Hope this helped you out!
