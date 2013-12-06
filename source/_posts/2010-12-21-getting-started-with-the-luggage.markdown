---
layout: post
title: "Getting Started with The Luggage"
date: 2010-12-21 09:47
comments: true
categories: ['luggage', 'os x', 'packaging']
---

With the current movement in the Mac Community toward modular imaging strategies, there's a spike in the need for properly formed package installers. Apple's package format is well documented for its benefits and flaws and there are a string of applications that will help you create your perfect package (from Apple's Packagemaker to the many third-party applications).  While all the various package creation applications will ultimately create a desirable package in the end, very few of them have the triple-threat of easy code-review, replication, and portability.  This is where The Luggage comes in.

[The Luggage](http://github.com/unixorn/luggage) is a packaging system based on GNU's make command that is found on most every *nix-based operating system (OS X included).  Joe Block, the author, open-sourced the project based on an internal Google tool for creating packages with Makefiles.  By using Makefiles, every member of your team can quickly glance at the file and see exactly what is being inserted into a package.  Changes can be made and committed quickly, errors can be squashed without tearing apart .pkg files, and you can reproduce packages quickly and efficiently without wading through many GUI windows.  

##Why do I need to Package?##
Many vendors already ship properly formatted package installers for their software - and for that we thank them.  There are still a couple of major software vendors that choose to use unique third-party package "wrappers" to install their software.  While this is fine if you only ever need to install that software on one machine, it makes software deployment...difficult.  Because of this, systems administrators need to re-package this software into a proper package installer that will deploy and install silently.  

If you're a fan of open-source software, you will find that many projects do not offer their software in Apple-friendly packages.  The Luggage will help you wrap their source files into a package format that can be distributed to your machines.

Finally, you may have a whole bevy of scripts that you use for configuration/customization of your clients.  These scripts can easily be deployed via ARD or wrapped into a payload-free package with a single postinstall script.  The Luggage will help you keep track of all your scripts and package them for distribution.

As I've said before, there are other third-party applications out there that will create a package to your needs.  Many will use snapshotting (or fseventsd monitoring) to create a package based on what's changed on your system.  While this is lightning fast (in most cases), you will need to redo this whole process if something needs to be changed in the resultant package.

##How do we use The Luggage?##

As we said before, The Luggage is just a giant Makefile.  Make has its own unique language, so you need to obey its syntax and formatting standards.  I've linked to the [GNU make manual here](http://www.gnu.org/software/make/manual/make.html) so you can get a quick overview of how it works (WARNING, it's quite large), but this guide will cover all the basics you need to know to get started.

Note that while it is **NOT NECESSARY TO COMPLETELY UNDERSTAND MAKE TO BEGIN USING THE LUGGAGE**, it will help you out tremendously as you start to create complicated packages if you DO understand make's nuances. This article may be long, but that's only to make sure that the reader understands what is going on in the background.

##The luggage.make File##

The base of everything you will do with The Luggage is a file called luggage.make.  It is by no means definitive, and you're encouraged to add to it should you encounter a situation where a recipe doesn't exist for a directory into which you want to install files (Don't worry, we'll get into this later), but it does serve as the basis for all packages you're going to be creating.

At the top of the file are all the variable declarations.  Anything that begins with SOMETHING=somethingelse is setting a variable that we will encounter later.  Many of these variables (such as PACKAGEMAKER, WORK_D, CP, INSTALL, and so on) are paths to various commands that we will need in our rules (setting absolute paths to common commands saves typing later and helps us avoid errors with things like PATH environment variables).  Dereferencing these variables in a Makefile is done with a syntax that looks like this -> ${CP} (this outputs the value of the CP variable, which is actually **/bin/cp**).  Note that you DO NOT use quotes when you set a variable (i.e. if you want to set the path to CP you do it with CP=/bin/cp and NOT by doing CP="/bin/cp") - if you DO use quotes, they will be included in the value of that variable (which will cause you all kinds of problems).

Next, we have the target stanzas.  A target (or a rule - I will use the word rule throughout the article) is setup to look like this:

``` make
            do-something: l_usr_bin
            @-echo "Commands are entered here"
            @-echo "Everything below our rule is executed"
```

In this case, the **target**, or **rule**, is called **do-something** and has dependencies based on ANOTHER rule that's called **l\_usr\_bin**.  Below this line is called our **recipe**, and right now there are two echo commands.  If the above code was in a blank text file called Makefile we could execute the two echo commands by running the following command from the command line:

``` bash
        make do-something
```


This would in turn echo the two lines of text below the do-something rule (This is not totally true - since we haven't defined a rule for l\_usr\_bin it would probably error out, but I've kept that dependency in the above example to show you how a rule works.).  Looking specifically at luggage.make, the target stanzas define the behavior for The Luggage's various behaviors (make pkg, make dmg, make zip, and so on).  Note that since many of these rules have dependencies on OTHER rules, a simple command of **make pkg** will trigger.

Next, you will encounter the Target directory rules.  This is the part of the luggage.make file that you may need to edit.  Joe has done a great job of defining the most popular directories into which you will install files/applications/etc. but it would be impossible for him to define EVERY location that you could possibly need.  Here is the structure of creating a Directory Rule:

``` make
            l_etc_puppet: l_etc
            @sudo mkdir -p ${WORK_D}/etc/puppet
            @sudo chown -R root:wheel ${WORK_D}/etc/puppet
            @sudo chmod -R 755 ${WORK_D}/etc/puppet
```


This rule will create an /etc/puppet directory in Luggage's working directory (The variable WORK_D will be used frequently - it's the temporary directory that The Luggage creates to simulate the target volume onto which it's installing files.  Everything following ${WORK\_D} will be the **EXACT PATH** into which files will be installed on the machine that runs your package.)  These Directory Rules become very important when we create custom Makefiles because they serve as dependencies which create the directories in The Luggage's Working Directory. In a nutshell, if you're installing files into a directory with The Luggage, you need to have a Directory Rule that creates this directory first.  Bottom Line: if you don't FIND a Directory Rule for a directory where you will be installing files, then you'll need to create one.

Finally are the File Packaging Rules.  These are handy shortcut rules that will keep your makefiles very short and readable.  In a nutshell, Joe has defined some of the most common directories to which files are installed and created one-line commands that will install specific files into those directories.  For example, say you were creating a custom Makefile in a directory that also had a launchd plist called **com.foo.bar.plist** in it.  If ALL you needed to do was create a package that installed that launchd plist into the /Library/LaunchDaemons directory you could setup your Makefile like this:

``` make
    include /usr/local/share/luggage/luggage.make

    TITLE=install_foo_launchd_plist
    REVERSE_DOMAIN=com.foo
    PAYLOAD=pack-Library-LaunchDaemons-com.foo.bar.plist
```


The PAYLOAD variable tells The Luggage which rules to execute.  In this case, it's executing the File Packaging Rule for launchd plists (pack-Library-LaunchDaemons-%) that creates the ${WORK_D}/Library/LaunchDaemons directory, installs the com.foo.bar.plist file into it, sets the correct permissions, and then executes the Packagemaker command line tool to create your package installer. Congratulations, you created a package in 5 lines of code!

##Preparing for using The Luggage##

The Luggage is fairly self contained, but it DOES use Apple's Packagemaker command line tool - and the way to install THAT is to download and install the Developer Tools for your computer's OS version (There are different Developer Tools packages for 10.6, 10.5, and so on).  The Developer Tools can be downloaded from [Apple's Developer Site](http://developer.apple.com), but you must create a (free) developer account first.  If you're a Mac sysadmin, you should already have all of this.

Once the Developer Tools have been installed you will then need to install The Luggage.  You can use The Luggage to create an installer package for The Luggage (Weird, yes) by first [downloading the source code from Github.](http://github.com/unixorn/luggage)  Just click on the big Downloads button in the upper right corner of the screen and download a zip version of the files.  From there, double-click on the downloaded zip file to open it, open up Terminal and change to the folder that was created which contains The Luggage's source code, and execute the following command:

``` bash
        sudo make pkg
```


This will create an installer package which can be run.  This package installs The Luggage into the /usr/local/share/luggage directory (Don't believe me? Check the Makefile for your self!).  You're now ready to use The Luggage!

Before we get into the package creation examples, we need to talk about directory structure and version control systems (svn, hg, git, etc...).  When you create new packages with The Luggage, it's necessary to create a new Makefile in a new directory.  I like to use the /usr/local/share/luggage/examples directory to create my packages.  The easiest way to begin a new package is to copy a directory that contains a working Makefile and simply tailor it to your needs.  Unless your job is SOLELY packaging (or you're a unix graybeard), you're NOT going to remember all the syntax and nuances of make.  Don't re-create the wheel - just copy and edit.  

Next, you're going to want to backup your files and/or have a versioning system.  Version control systems (like Subversion, Mercurial, Git, and the like) are becoming increasingly popular [with the current DevOps movement](http://www.agileweboperations.com/the-state-of-devops), so it might be a good idea to start playing with one NOW while you're learning a new skill!  If you're TOTALLY new to this concept, [I recommend using git](http://help.github.com/mac-git-installation/), but it's entirely up to you.  [I maintain a git repository of my Luggage fork](http://github.com/huronschools/luggage) that's open to anyone to review and borrow.  If you decide to use something like Git, well then GOOD ON YA, MAN!  If not, make sure you have a backup of your Makefiles.  They're small files; make two backups :)

##Package Creation Examples##

Now that we've described how make works, how luggage.make works, given a quick example of how to create a package in 5 lines, and had the installation/backup talk let's walk you through some basic package creation examples. We'll create packages that install an application, create a package that installs printers and executes a postinstall script, create a package that installs source code in many directories, and finally create a complex Makefile that uses variables and bash commands.

##An Application Package##

The most common packages will simply install an application file into the /Applications folder on your computer.  Since .app files are actually bundles (a directory that contains subdirectories of executable code), we will need to use tar to wrap these bundles into a single file.  From there we can have The Luggage untar the application into the correct folder. [Joe's Firefox example](https://github.com/unixorn/luggage/blob/master/examples/firefox/Makefile) does this, but he also has an added step of curling the tarred/bzipped application from a web server.  I'm going to skip that step (for now) so you understand how the basic process works, but it's a best practice to use the curl method so you can keep your applications up-to-date on your fileserver without having to copy the new files to your luggage directory every time.  

Let's first create the /usr/local/share/luggage/examples/calculator directory and then make a copy of /Applications/Calculator.app into the /usr/local/share/luggage/examples/calculator directory.  Next, lets open Terminal and change to our /usr/local/share/luggage/examples/calculator directory.  Finally, run the following command to tar.bz2 our Calculator.app:

``` bash
        tar cvfj Calculator.app.tar.bz2 Calculator.app
```


If you did it correctly, it should create the Calculator.app.tar.bz2 file that we need for The Luggage.  Make sure this file is in the SAME directory (and level - so don't create subfolders) as the Makefile we're going to create.  You can actually delete the Calculator.app copy that we created - we won't need it.  From there, our Makefile is simple - it should look like this:

``` make
    include /usr/local/share/luggage/luggage.make

    TITLE=Calculator_Install
    REVERSE_DOMAIN=com.yourdomain
    PAYLOAD=unbz2-applications-Calculator.app
```


That's it!  The PAYLOAD variable is the magic variable here - it contains the rule(s) that are to be executed in creating our package.  The unbz2-applications-% File Packaging Rule is defined in our luggage.make file (which we've included at the top of our Makefile), so we don't NEED anything else in our Makefile.  **MAKE SURE** that the capitalization and spelling of Calculator.app in "unbz2-applications-Calculator.app" and your "Calculator.app.tar.bz2" files are IDENTICAL or this package will NOT be created (make relies on this).  

##UPDATED: An easier Application Package##

Joe has actually created a Ruby script that will perform all of the actions outlined in the previous example for you.  It's called app2luggage.rb and it can save you a few steps if you know how to use it.  Let's take a look and see what we need to use it.

The first thing you will need to do is install the trollop rubygem as that's what app2luggage.rb uses to parse its options.  You can do this with the following command:

``` bash
        sudo gem install trollop
```


Next, make sure the app2luggage.rb script exists in your /usr/local/share/luggage directory.  Finally, let's run the command with the following arguments:

``` bash
        sudo /usr/local/share/luggage/app2luggage.rb -a /Applications/Calculator.app -i /usr/local/share/luggage/examples/Calculator_Application/ -l /usr/local/share/luggage/luggage.make -r com.huronhs -p Calculator_Application
```


Here's what each argument means:

1. -a is the path to the Application we want to tar up and install with The Luggage - we're using the path to Calculator.app for now
2. -i is the path to the folder that app2luggage.rb will create that contains our Makefile and tarred up application.  This folder SHOULD NOT EXIST or app2luggage.rb will exit (so as not to overwrite your data).
3. -l is the path to our luggage.make file
4. -r is the reverse domain for our organization
5. -p is the package id for the Makefile

There are other arguments available, simply run app2luggage.rb with the --help argument to see them all.  Once app2luggage.rb runs successfully, it will create the directory you specified in the -i argument and populate it with a Makefile and the tarred up application.  The only thing left to do is to make your pkg, zip, or dmg.

##Installing a printer with a preinstall script##

One of the most popular solutions I offer with using The Luggage is to create a package that will install a printer and then install a pre-configured PPD file that sets all the initial settings for the printer.  I do this with Puppet currently, and I know it's popular in Munki too.  You can also optionally install a specific driver (if the drivers for your printer aren't already on your machines).  For those people who like to skip ahead, [I have this example on my luggage repo](https://github.com/huronschools/luggage/tree/master/examples/hhs_printers/Printserver%20Managed/Office_9050).  

This Makefile demonstrates the use of a preinstall/preflight script (which isn't actually installed into the payload of the package).  The Luggage has a special packaging rule for this called pack-script-% that works so long as the name of your script corresponds exactly with what you write in the PAYLOAD variable.  This file also demonstrates the use of multiple rules in the PAYLOAD variable by using the \ character.  While this isn't difficult to understand, it is a necessary syntax.

[My preflight script is right here](https://github.com/huronschools/luggage/blob/master/examples/hhs_printers/Printserver%20Managed/Office_9050/preflight) for those who are interested.  It's copied into the directory we create that contains our makefile and I name it preflight.  Note that it contains variables that need to be changed (all of which are at the top of the script) before you deploy this package.  

Next, we need to get a .ppd file that contains all the configuration data for our printer.  The easiest way to do this is to install your printer on a demo machine using the EXACT SAME SETTINGS that will be configured in your script (protip: actually RUN the script on your computer first), configure it how you want (number of trays, memory settings, type of finisher, etc...), and then open the /etc/cups/ppd directory on your model computer.  Inside should be a .ppd file for your printer containing all the settings you've just configured.  Copy this .ppd file into the folder that contains your Makefile and preflight script.  Mine (in this example) will be called psm\_HHS\_Office\_9050.ppd

Now, let's look at our Makefile:

{% codeblock Managed Printer Makefile lang:make %}
    include /usr/local/share/luggage/luggage.make

    TITLE=HHS_Main_Office_9050_Managed_Installer
    REVERSE_DOMAIN=com.huronhs
    PAYLOAD=\
        pack-hp-ppd \
        pack-script-preflight

    pack-hp-ppd: l_etc_cups_ppd
      @sudo ${CP} ./psm_HHS_Office_9050.ppd ${WORK_D}/etc/cups/ppd/psm_HHS_Office_9050.ppd
      @sudo chmod 644 ${WORK_D}/etc/cups/ppd/psm_HHS_Office_9050.ppd
      @sudo chown root:_lp ${WORK_D}/etc/cups/ppd/psm_HHS_Office_9050.ppd
    
{% endcodeblock %}

The first thing you should notice is that our PAYLOAD variable starts with a \ and has three lines.  The \ signifies that the contents of our variable spans multiple lines.  In reality, the value of PAYLOAD is actually "pack-hp-ppd pack-script-preflight" but it's formatted so it's easier to read.  Next, notice that the pack-script-preflight rule contains the word "preflight" after pack-script-.  This means that our script must be named preflight (EXACTLY - case is sensitive here).  Finally, we've also specified a pack-hp-ppd rule.  Since luggage.make DOES NOT define this rule, we've defined it in our Makefile.  

The pack-hp-ppd rule has a dependency on the l\_etc\_cups\_ppd rule.  This rule IS defined in luggage.make (well, it is for me - I can't remember if I created it or not.  If it isn't there for you, then you'll need to create it using the other folder creation rules as a guide) - and it creates the /etc/cups/ppd folder structure that we need in our package.  Lets look at the three lines which are called our **recipe***:

``` make
    @sudo ${CP} ./psm_HHS_Office_9050.ppd ${WORK_D}/etc/cups/ppd/psm_HHS_Office_9050.ppd
    @sudo chmod 644 ${WORK_D}/etc/cups/ppd/psm_HHS_Office_9050.ppd
    @sudo chown root:_lp ${WORK_D}/etc/cups/ppd/psm_HHS_Office_9050.ppd
```


The first line references the CP variable (if you remember - its value is /bin/cp) to copy the psm\_HHS\_Office\_9050.ppd file from the directory that contains our Makefile into Luggage's working directory/etc/cups/ppd.  The second line sets its mode to 644 (to match the mode for all .ppd files in that directory), and the third line sets the owner to root and the group to _lp (note that this group is ONLY available in 10.5 and 10.6 machines - so you'll need another package for 10.4 clients or below).  

That's it!  The resultant package will run the preflight script and then install the correct .ppd file after the script is run.  This package will successfully install and configure your printer in one fell swoop.

##Installing files into multiple directories##

So far our Makefiles have been pretty easy.  In fact, most of them have been a couple of lines.  Lets look at a package that installs files into multiple locations.  There's a cool open source tool called [Angelia that was created by R.I Pienaar](http://github.com/ripienaar/angelia).  I use it to send alerts to my iPhone and to also process Nagios Alerts.  The problem is that it's only packaged for Linux machines.  Since I manage Mac machines, I thought I'd lend R.I. a hand and create a package installer for it.  [This package is also on my luggage repo](https://github.com/huronschools/luggage/tree/master/examples/angelia) if you want to work ahead.  Let's look at the Makefile:

{% codeblock Angelia Recipe lang:make %}
    #  Angelia Package Creation:

    include /usr/local/share/luggage/luggage.make

    TITLE=Angelia_Installer
    REVERSE_DOMAIN=com.huronhs
    PAYLOAD=\
        pack-angelia-etc \
        pack-angelia-binaries \
        pack-angelia-lib \
        pack-angelia-spool \
        pack-angelia-log \
        pack-angelia-launchd


    pack-angelia-launchd: l_Library_LaunchDaemons
      @sudo ${CP} net.devco.angelia.plist ${WORK_D}/Library/LaunchDaemons
      @sudo chmod -R 644 ${WORK_D}/Library/LaunchDaemons

    pack-angelia-binaries: l_usr_sbin
      @sudo ${CP} ./angelia-send.rb ${WORK_D}/usr/sbin/angelia-send
      @sudo ${CP} ./angelia-spoold.rb ${WORK_D}/usr/sbin/angelia-spoold
      @sudo ${CP} ./angelia-nagios-send.rb ${WORK_D}/usr/sbin/angelia-nagios-send
      @sudo chmod -R 755 ${WORK_D}/usr/sbin

    pack-angelia-lib: l_Library_Ruby_Site_1_8
      @sudo ${CP} -R ./angelia ${WORK_D}/Library/Ruby/Site/1.8
      @sudo ${CP} ./angelia.rb ${WORK_D}/Library/Ruby/Site/1.8
      @sudo chmod -R 755 ${WORK_D}/Library/Ruby/Site/1.8

    pack-angelia-etc: l_etc_angelia
      @sudo ${CP} -R ./templates ${WORK_D}/etc/angelia
      @sudo ${CP} ./angelia.cfg ${WORK_D}/etc/angelia
      @sudo ${CP} ./COPYING ${WORK_D}/etc/angelia
      @sudo ${CP} ./README.markdown ${WORK_D}/etc/angelia
      @sudo chmod -R 755 ${WORK_D}/etc/angelia

    pack-angelia-log: l_var_log_angelia
      @sudo touch .create
    pack-angelia-spool: l_var_spool_angelia
      @sudo touch .create
    
{% endcodeblock %}

It's definitely longer, but I don't think it's any more complex than the examples we've seen before.  The PAYLOAD variable spans multiple lines using the \ character and I've defined all the custom rules.  All of the rules depend on [folder rules in my luggage.make file](https://github.com/huronschools/luggage/blob/master/luggage.make) (which were created if they didn't exist before), and the recipes inside those rules are extremely simple to navigate (mainly copying files and changing permissions).  Creating a script that makes this package would be QUITE A BIT longer than this Makefile, and the Makefile itself took less than 3 minutes to create.  The only caveat is that we need to make sure all the files/directories we're copying into Luggage's WORK_D directory are in the folder that contains the Makefile.  Because of this, anytime Angelia is updated I will need to copy over new files.  

It would be much easier to create a Makefile that downloaded the current version of all of these files before it copied them into our package...

##Creating a more complex Makefile##

The basis of this example was my thought that it would be MUCH easier for me to have a Makefile that downloaded the newest versions of the files that I need before it copies them into the package.  Make supports this through its support of shell commands and variables - it only requires you to code it up.  Let's look at a package I created for Marionette-Collective.

[Marionette-Collective is awesome software](http://github.com/puppetlabs/marionette-collective) (also created by R.I Pienaar and now owned by Puppet Labs) that allows you to communicate with multiple nodes at once.  They DID have a script that created their packages, but I wanted to create a Makefile that would create a package of ANY version of their software.  Let's look at the Makefile first and I'll break it down:

{% codeblock MCollective Recipe lang:make %}
    #  Example:  A dynamic installer for Marionette-Collective
    #
    #  Author:  Gary Larizza
    #  Created: 12/17/2010
    #  Last Modified: 12/18/2010
    #
    #  Description:  This Makefile will download the version of MCollective specified
    #    in the PACKAGE_VERSION variable from the Puppet Labs website, untar it,
    #    and then install the source files into their Mac-specific locations.  
    #    The MAJOR and MINOR versions must be specified for the Info.plist file
    #    that Packagemaker requires, but I use awk on the PACKAGE_VERSION to 
    #    get these.  See inline comments.
    #
    include /usr/local/share/luggage/luggage.make


    # Luggage Variables:
    #    If the TYPE variable isn't specified via the CLI, we will install everything
    #    into the resultant package
    TITLE=MCollective_Installer_Full
    REVERSE_DOMAIN=com.puppetlabs
    PAYLOAD=\
        unpack-mc-${MCFILE} \
        pack-mc-libexec \
        pack-mc-binaries \
        pack-mc-lib \
        pack-mc-config \
        pack-mc-config-server \
        pack-mc-config-client \
        pack-mc-launchd \
        pack-mc-mcollectived \
        pack-mc-preflight-all

    # Variable Declarations:  
    #    Any variable can be set from the command line by doing this:
    #    "make pkg PACKAGE_VERSION=1.0.0"
    PACKAGE_VERSION=1.0.0
    PACKAGE_MAJOR_VERSION=`echo ${PACKAGE_VERSION} | awk -F '.' '{print $$1}'`
    PACKAGE_MINOR_VERSION=`echo ${PACKAGE_VERSION} | awk -F '.' '{print $$2$$3}'`
    MCFILE=mcollective-${PACKAGE_VERSION}
    MCURL=http://puppetlabs.com/downloads/mcollective/${MCFILE}.tgz

    # Package Creation Limiters:
    #    These if-statements will check for one of three values for the TYPE variable:
    #    "COMMON, CLIENT, or BASE"  If either of these values are present (CASE SENSITIVE)
    #    the PAYLOAD variable will be changed to limit what is installed into the package.

    # COMMON Package:
    #    This package includes the Ruby libraries and MCollective plugins with nothing else.
    ifeq (${TYPE},COMMON)
      PAYLOAD=\
          unpack-mc-${MCFILE} \
          pack-mc-libexec \
          pack-mc-lib \
          pack-mc-preflight-common
      TITLE=MCollective_Installer_Common
    endif

    # CLIENT Package:
    #    This package includes the MCollective Binaries and the configuration file for
    #    MCollective's client binaries.  
    ifeq (${TYPE},CLIENT)
      PAYLOAD=\
          unpack-mc-${MCFILE} \
          pack-mc-config \
          pack-mc-binaries \
          pack-mc-config-client \
          pack-mc-preflight-client
      TITLE=MCollective_Installer_Client
    endif

    # BASE Package:
    #    This package includes the mcollectived daemon, Ruby Libraries, a launchd plist
    #    to call mcollectived, and the configuration files for the MCollective server.
    ifeq (${TYPE},BASE)
      PAYLOAD=\
          unpack-mc-${MCFILE} \
          pack-mc-config \
          pack-mc-config-server \
          pack-mc-launchd \
          pack-mc-mcollectived \
          pack-mc-lib \
          pack-mc-preflight-base
      TITLE=MCollective_Installer_Base
    endif

    # This rule will curl the selected version of MCollective and untar it into the directory
    #    in which the Makefile resides.
    unpack-mc-${MCFILE}:
      curl ${MCURL} -o ${MCFILE}.tgz
      @sudo ${TAR} xzf ${MCFILE}.tgz

    # This rule will install MCollective's plugin files to /usr/libexec/mcollective
{% endcodeblock %}
