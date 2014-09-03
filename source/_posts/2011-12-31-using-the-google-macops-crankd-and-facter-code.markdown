---
layout: post
title: "Using the Google Macops Crankd and Facter Code"
date: 2011-12-31
comments: true
categories: 
---


On December 21st, the MacOps team at Google released a Googlecode repository containing a couple of scripts that they use for managing Macs.  The scripts provided interface with [crankd (available in the Pymacadmin code suite)](http://code.google.com/p/pymacadmin/) and [Facter](http://puppetlabs.com/puppet/related-projects/facter/), which are a couple of tools with which you MIGHT not be familiar.

I saw a couple of questions on the ##osx-server IRC channel about how these scripts work, and I decided to do a quick writeup on how you would implement the scripts in your environment.  This is meant to be a walkthrough to getting their scripts up-and-running, but please READ EVERYTHING FIRST before implementing.


##Prepare Crankd:

Crankd is a vital part of the Google MacOps puzzle.  Crankd essentially executes Python code in response to System/Network events on your system.  I have MUCH MORE information on crankd [in a guide I posted a while back](http://glarizza.posterous.com/using-crankd-to-react-to-network-events), but I'm going to give you the 'quick version' for those looking to get started quickly:

* Clone or download the Pymacadmin Github repo.  If you have git installed you can accomplish that by doing the following from the command line:

```
git clone https://github.com/acdha/pymacadmin.git
```

If you DON'T have git on your machine just visit [https://github.com/acdha/pymacadmin](https://github.com/acdha/pymacadmin), click on the Downloads tab, and download the .tar.gz or .zip version of the repo.  Next, double-click on the resultant file to expand it into a folder.

* Change to the pymacadmin folder and run the install-crankd.sh script which will install crankd.py into /usr/local/sbin and create /Library/Application Support/crankd  Once you've changed to the pymacadmin source directory, run the following from the command line:

```
sudo ./install-crankd.sh
```

## OR...

If you already have a Puppet environment configured, I've written a [module that will install crankd for you.](http://github.com/glarizza/puppetlabs-crankd) The upside is that this module should work out of the box, but the downside is that I've copied the source to the module's files/ directory.  Since Pymacadmin hasn't been changed since 2009, though, this shouldn't pose a big issue.  Simply copy this module to your modulepath and declare it with the following line in your node declarations:

{% codeblock lang:puppet %}
include crankd
{% endcodeblock %}

##Download the Google-macops source code

You can download the source by using subversion with the following command:

```
svn checkout http://google-macops.googlecode.com/svn/trunk/ google-macops-read-only
```

This will create a folder called 'google-macops-read-only' in the current directory.

##Copy the Python crankd files

Change to the directory where you downloaded the Google-macops source code and you will notice two folders: crankd and facter.  Change to the crankd directory and copy all of the python files to /Library/Application Support/crankd with the following command from the command line:

```
sudo cp *.py /Library/Application\ Support/crankd/
```

This copies two files into /Library/Application Support/crankd: ApplicationUsage.py and NSWorkspaceHandler.py.  The ApplicationUsage.py file contains most of the magic.  In a nutshell, it will create a SQLite database file called /var/db/application\_usage.sqlite containing information on applications that are launched and quit (we will inspect this information later), and it will update the database any time an application is (you guessed it) launched or quit.  Crankd provides the mechanism by which the code is run any time an application is launched/quit and these Python files actually update the database with the pertinent information.

##Copy the provided crankd plist to /Library/Preferences

Changing to the crankd directory in the google-macops-read-only directory you should find a file called 'com.googlecode.pymacadmin.crankd.plist' which is a sample plist for utilizing Google's code that tracks application usage via crankd.  It's up to you to merge these Python methods into your existing crankd setup, or utilize the plist they've provided.  The plist essentially says that any time an application is LAUNCHED, you should call the 'OnApplicationLaunch' method in the NSWorkspaceHandler class, and any time an application is QUIT, you should call the 'OnApplicationQuit' method in the NSWorkspaceHandler class.  Let's accept these conditions and copy their plist into /Library/Preferences (assuming that we don't have an existing crankd setup) with the following command:

```
cp com.googlecode.pymacadmin.crankd.plist /Library/Preferences
```

##Test out crankd from the command line

With crankd.py installed, the Python methods copied to /Library/Application Support/crankd, and the com.googlecode.pymacadmin.crankd.plist plist copied to /Library/Preferences, we can finally test out the code to see how it works.  Start crankd with the following command:

```
sudo /usr/local/sbin/crankd.py --config=/Library/Preferences/com.googlecode.pymacadmin.crankd.plist
```

After entering your password, you should see the following in your terminal:

```
INFO: Loading configuration from /Library/Preferences/com.googlecode.pymacadmin.crankd.plist
INFO: Listening for these NSWorkspace notifications:  NSWorkspaceWillLaunchApplicationNotification, NSWorkspaceDidTerminateApplicationNotification
```

Now, let's launch a couple of applications and see how we track their usage through the sqlite database.  Do this by starting Safari and TextEdit, and then closing them (note that you MUST start them fresh - if they're already opened make sure to close them, then open them, and THEN close them again).  You should see something that looks like the following in the terminal:

```
INFO: Application Launched: bundle_id: com.apple.Safari version: 6534.52.7 path: /Applications/Safari.app
INFO: Application Launched: bundle_id: com.apple.TextEdit version: 264 path: /Applications/TextEdit.app
INFO: Application Quit: bundle_id: com.apple.TextEdit version: 264 path: /Applications/TextEdit.app
INFO: Application Quit: bundle_id: com.apple.Safari version: 6534.52.7 path: /Applications/Safari.app
```

Finally, you can quit crankd from the terminal by holding the control button down and pressing the 'c' key on your keyboard.

##Inspect the application\_usage.sqlite database

If you're proficient in SQL, feel free to browse through the contents of the database.  To keep things simple, let's download an app called [SQLite Database Browser](http://sourceforge.net/projects/sqlitebrowser/files/latest/download) from the following link.  Launch SQLite Database Browser, go to File -&gt; Open Database, hold the Shift and Command buttons down on your keyboard and then press the 'g' button to bring up the 'Go to the folder:' dialog box, type in /var/db/application_usage.sqlite in the box and press return, and then finally click the open button to open the database.  If you click the 'Browse Data' tab near the top of the window, you should be able to see the raw data in the database.  It should list the last time (in epoch time), bundle id, version, path, and number of times that the application has been launched (note that you can use an [online utility](http://www.onlineconversion.com/unix_time.htm) to convert from epoch time to regular time).

##Inspect the Facter fact

[If you're not familiar with Facter](http://puppetlabs.com/puppet/related-projects/facter/), you should click on the link and check it out.  Facter is an awesome binary designed to gather information about your system and report it back in the form of a 'fact', which is essentially a key =&gt; value pair (for example, the 'operatingsystem' fact will return a value of 'Darwin' on Macs running OS X or 'RedHat' on machines running RedHat Linux).  Google has provided a custom Facter fact that will report a fact NAME that corresponds with an application you've launched on your system (such as 'app_lastrun_com.apple.safari' or 'app_lastrun_com.apple.textedit') and a VALUE with the epoch time or the last time that application was run.  The output will ultimately look like this:

```
app_lastrun_com.apple.safari => 1325203354
app_lastrun_com.apple.textedit => 1325203357
```

**NOTE:** This Facter fact requires the SQLite database that is created by the crankd code from above, so make sure you've followed the previous steps or you might have problems getting fact values.

We'll need to install Facter.  [You can download a copy of Facter from Puppet Labs directly](http://downloads.puppetlabs.com/mac).  Simply download the DMG, expand it, and run the package installer inside.

To test out the fact, we will need the full path to the folder containing the google-macops code we downloaded in step two above.  I downloaded the source to a path of /Users/gary/Desktop/google-macops-read-only and so that's the path I'm going to use in the next step.

Facter is designed to work with [Puppet](http://puppetlabs.com/puppet/puppet-enterprise/), but you can also run it independently.  Facter is ALSO designed to be plug-able with custom facts, and that's what Google has shipped us.  In order to TEST these custom facts, we need to set the RUBYLIB environment variable (as Facter will look for a custom facts inside a 'facter' directory that's located inside the path returned by the RUBYLIB environment variable).  RUBYLIB should be set to the path of the downloaded google-macops source, so in my case I would run the following from the command line:

{% codeblock lang:bash %}
export RUBYLIB=/Users/gary/Desktop/google-macops-read-only
{% endcodeblock %}

**NOTE:**  make sure RUBYLIB is in capital letters.  If you downloaded the google-macops source to a different path, substitute your path for '/Users/gary/Desktop/google-macops-read-only'.

Next, we need to make sure we have the 'sqlite3' ruby library installed on our system as the custom fact requires this library for correct operation.  One way to install sqlite3 is by using Rubygems with the following command from the command line:

```
sudo gem install sqlite3
```

**NOTE:** If you have troubles using Rubygems to install sqlite3, you can either build it from source (which is beyond the scope of this article), or you can use Macports (if you have it installed) to install it with the following command:

```
sudo port install sqlite3
```

The last step is to actually run Facter to test out the fact.  You may have a problem TESTING this fact if you installed sqlite3 via Rubygems - the reason is because the custom fact doesn't actually load Rubygems before it tries to load sqlite3 (this isn't a problem if you use the fact with Puppet as Puppet loads Rubygems.  It's ALSO not a problem if you're running version 1.9 or greater of ruby, as you no longer need to load Rubygems BEFORE trying to include a ruby library with ruby 1.9 or greater).  To remedy this problem, let's add a single line to the custom fact.  Notice the line that says the following:

{% codeblock lang:ruby %}
require 'sqlite3'
{% endcodeblock %}

This line is where the custom fact loads the sqlite3 library.  Now, the line BEFORE this line we need to add a single line:

{% codeblock lang:ruby %}
require 'rubygems'
{% endcodeblock %}

This line will load Rubygems so that it knows where to access the sqlite3 library.  Note that this line is unnecessary if you use this custom fact with Puppet as, like I mentioned before, Puppet already loads Rubygems.  For the purpose of our testing, though, we will need this line.

Finally, let's test the fact by running Facter from the command line:

```
facter
```

You SHOULD get a list of ALL your Facter facts, but up at the top you should have a couple of facts that begin with 'app_lastrun_' and then end with the bundle\_id of the application that was launched.  These are the custom facts that have been created.

You'll note that the time reported is in epoch time.  If you'd prefer a date/time string that's more legible, you'll need to modify the apps.rb file in the facter directory.  Line 39 in the file (or, line 40 if you added the require 'rubygems' line previously) should say 'appdate' but change it to say the following:

{% codeblock lang:ruby %}
Time.at(appdate.to_i)
{% endcodeblock %}

Running facter again (like above) will give you a more legible date:

```
app_lastrun_com.apple.safari => Thu Dec 29 19:02:34 -0500 2011
app_lastrun_com.apple.textedit => Thu Dec 29 19:02:37 -0500 2011
```

 
##???
##Profit!

Now that you know how the code works, you're free to modify, extend, and improve it as you see fit!  The one thing that's missing here is a LaunchDaemon that keeps crankd running in the background.  Since you're maintaining your environment, I'll leave that up to you (though I walk you through this step in ['Step 5.  A LaunchDaemon to keep crankd running' in my previous crankd writeup](http://glarizza.posterous.com/using-crankd-to-react-to-network-events)).  Feel free to check out my other writeups on Facter, Puppet, and Crankd to see how I used to manage hundreds of Macs in an educational environment!
