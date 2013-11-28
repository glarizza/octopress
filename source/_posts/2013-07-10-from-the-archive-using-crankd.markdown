---
layout: post
title: "From the archive: Using Crankd"
date: 2013-07-10 09:51
comments: true
categories: 
---

Supporting laptops in a managed environment is tricky (and doubly so if you allow them to be taken off your corporate network).  While you can be reasonably assured that your desktops will remain on and connected during the workday, it's not uncommon for laptops to go to sleep, change wireless access points, and even change between an Ethernet or AirPort connection several times during the day.  It's important to have a tool that can "tweak" certain settings in response to these changes.

This is where crankd comes in.

Crankd is a cool utility that's part of the Pymacadmin (http://code.google.com/p/pymacadmin/) suite of tools co-authored by Chris Adams and Nigel Kersten.  Specifically crankd is a Python daemon that lets you trigger shell scripts, or execute Python methods, based upon state changes in SystemConfiguration, NSWorkspace and FSEvents.  

##Use Cases##

It's easier to see how crankd can help you with a couple of scenarios:

1.  Your laptops, like all of the other machines in your organization, are bound to your corporate LDAP servers.  When they're on network, they will query the LDAP servers for things like authentication information.  Unless your corporate LDAP directory is accessible outside your corporate network, your laptops may exhibit the "spinning wheel of death" when they attempt to contact a suddenly-unreachable LDAP directory at the neighborhood Starbucks.  A solution to this is to remove the LDAP servers from your Search (and Contacts) path whenever the laptop is taken off-network and add the LDAP servers when you come back on-network.

2.  Perhaps you're using Puppet, Munki, Chef, StarDeploy, Filewave, Absolute Manage, Casper, or any other configuration management system that needs to contact a centralized server for configuration information. Usually these tools will have your machine contact their servers once an hour or so, but this can be a problem if the machine is constantly sleeping and waking.  Plus if you take your machine off-network, you don't want it trying to contact a server that might not be reachable from the outside world. It would be nice to have your laptop "phone home" when it establishes a network connection on your corporate network, and skip this step when the laptop is taken outside your organization.

3.  OS X allows you to set a preferred order for your network connections, but it would be nice to disable the AirPort when your laptop establishes an Ethernet connection.

4.  Finally, maybe you have the need to perform an action whenever your laptop sleeps (or wakes), changes a network connection, mounts a volume, or runs a specific Application (whether it's located in the Applications directory or anywhere else on your machine).

All of these situations can be made trivial through the help of crankd.

##How do I get it working?##

Crankd is a daemon, so it's running in the background while you work.  It uses an XML plist file that tells it which scripts (or which Python methods) to execute in response to specific state changes (like a network connection going up or down or a volume being mounted).  Since it's a small Python library, the files aren't huge and the entire finished installation is around 100 Kb (or larger with your custom code/scripts).  Lets download crankd and experiment with its settings:

1.  **Download the Pymacadmin source.**  You can do this through Google Code or Github - I'll demonstrate the Github method.  Navigate to http://github.com/acdha/pymacadmin, click the Downloads button, and download either the .tar.gz or the .zip version of the source code.  Drag it to your desktop and then double-click on the file to expand it.  It should open a folder named "acdha-pymacadmin-<combination of 7 numbers and letters>"

2.  **Install crankd**  Upon opening the pymacadmin folder, you should see a series of folders, readme files, and an "install-crankd.sh" installation script.  Let's open Terminal.app and navigate to the pymacadmin folder that we expanded on our desktop (you can type "cd" into Terminal.app and then drag and drop the folder into the Terminal window.  Hit the Return button on your keyboard to change to the directory.).  The install-crankd.sh script is executable, so run it by typing "sudo ./install-crankd.sh" into the Terminal window and hitting Return.  Enter your password when it prompts you

3.  **Setup a plist file for crankd**  If you've never worked with crankd before, it's best to let it setup a configuration plist for you.  If you don't specify a configuration plist with the "--config" argument, or you don't have a com.googlecode.pymacadmin.crankd.plist file in your /Users/<username>/Library/Preferences folder, crankd will automatically create a sample plist for you.  Let's do that by typing "/usr/local/sbin/crankd.py" into Terminal and hitting the Return button.  Take a look at the sample configuration plist file:
  
  <?xml version="1.0" encoding="UTF-8"?>
  <!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
  <plist version="1.0">
  <dict>
    <key>NSWorkspace</key>
    <dict>
      <key>NSWorkspaceDidMountNotification</key>
      <dict>
        <key>command</key>
        <string>/bin/echo "A new volume was mounted!"</string>
      </dict>
      <key>NSWorkspaceDidWakeNotification</key>
      <dict>
        <key>command</key>
        <string>/bin/echo "The system woke from sleep!"</string>
      </dict>
      <key>NSWorkspaceWillSleepNotification</key>
      <dict>
        <key>command</key>
        <string>/bin/echo "The system is about to go to sleep!"</string>
      </dict>
    </dict>
    <key>SystemConfiguration</key>
    <dict>
      <key>State:/Network/Global/IPv4</key>
      <dict>
        <key>command</key>
        <string>/bin/echo "Global IPv4 config changed"</string>
      </dict>
    </dict>
  </dict>
  </plist>
  
This XML file has two main keys - one for NSWorkspace events (such as mounted volumes and sleeping/waking your laptop), and one for SystemConfiguration events (such as network state changes).  Followed by a key for the specific event that we're monitoring, a key specifying whether we'll be executing a command or a Python method in response to this event, and a string (or an array of strings, as we'll see later) specifying the actual command that's to be executed.  For all of the events in the sample plist, we're going to be echoing a message to the console.

4.  **Start crankd**  Once crankd has been installed and your configuration plist file is setup, you're ready to let crankd monitor for state changes. Let's start crankd with the sample plist that was created in the previous step by executing the following command in Terminal "/usr/local/sbin/crankd.py --config=/Users/<username>/Library/Preferences/com.googlecode.pymacadmin.crankd.plist"  Remember to substitute your username for <username> in that command (if you don't know your username, you can type "whoami" into Terminal and hit the Return button).  If everything was executed correctly, you should see the following lines displayed in Terminal:
  
  Module directory /Users/<username>/Library/Application Support/crankd does not exist: Python handlers will need to use absolute pathnames
  INFO: Loading configuration from /Users/<username>/Library/Preferences/com.googlecode.pymacadmin.crankd.plist
  INFO: Listening for these NSWorkspace notifications: NSWorkspaceWillSleepNotification, NSWorkspaceDidWakeNotification, NSWorkspaceDidMountNotification
  INFO: Listening for these SystemConfiguration events: State:/Network/Global/IPv4
  
It might look like Terminal isn't doing anything, but in all actuality crankd is listening for changes.  You can make crankd come to life by either connecting to (or disconnecting from) an AirPort network, sleeping/waking your machine, or mounting a volume (by inserting a USB memory stick, for example).  Performing any of these actions will cause crankd to echo messages to your Terminal window.  Here's the message I received when I disconnected from an AirPort network:

  INFO: SystemConfiguration: State:/Network/Global/IPv4: executing /bin/echo "Global IPv4 config changed"
  Global IPv4 config changed
  
To quit this sample configuration of crankd, simply hold down the control button on your keyboard and press the C key.  Congratulations, crankd is now up and running!

##A more complex example##

Let's look at one of our previous situations

  

