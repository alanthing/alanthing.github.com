---
layout: post
title: "Dropbox CLI for CentOS 5 the easy way"
date: 2010-08-12 12:27
comments: true
categories:
---

*This post originally featured on the [Echo &amp; Co. blog](http://echo.co/blog/dropbox-cli-centos-5-easy-way)*.

[Dropbox](http://www.dropbox.com/) hardly needs any introduction; put files in your Dropbox and they show up everywhere else you have Dropbox installed and dropbox.com. A feature about Dropbox that is probably not as widely known is that free accounts come with 30 days of undo history and Pro accounts can get "Pack Rat" that keeps unlimited history of changes. The history of files, including reverting deleted files, was particularly interesting to me, since I could hook in my latest daily MySQL dumps from [AutoMySQLBackup](http://sourceforge.net/projects/automysqlbackup/) to Dropbox and have 30 days of backups for free available from anywhere dropbox.com is accessible. 

The problem is that we use [CentOS](http://centos.org/) for our servers and the Dropbox Linux builds are geared for distributions like Ubuntu and Debian that have updated versions of required software like Python, libc, and others, that I did not want to upgrade by hand on my systems and risk the integrity of the system packages. But, I got it to work anyway, read on for how I got Dropbox CLI installed on CentOS without replacing any system files.

*	Download http://www.getdropbox.com/download?plat=lnx.x86 or http://www.getdropbox.com/download?plat=lnx.x86_64 [(ref)](http://forums.dropbox.com/topic.php?id=8386#post-53456)

*	Extract tar.gz file downloaded and leave in ~ of desired user

*	Run `~/.dropbox-dist/dropboxd` to get Dropbox to provide a URL to go to in your browser to link this computer to your Dropbox account

```bash
$ ./dropboxd
This client is not linked to any account...
Please visit https: //www.dropbox.com/cli_link?host_id=xxxxxxxxxxxxxxxxxxxxxxxxxxxx to link this machine.
```

*	After visiting the URL in a browser to which you've logged into dropbox.com, you'll see the following output:

```bash
/usr/bin/nautilus
cannot open display: 
Run 'nautilus --help' to see a full list of available command line options.
```

*	If you cannot quit the app, open another a shell, get the PID by running `ps -ef|grep dropbox`, and kill PID. The output on your other shell should say:
Terminated

*	Download the official Dropbox CLI: http://www.dropbox.com/download?dl=packages/dropbox.py [(ref)](http://wiki.dropbox.com/TipsAndTricks/TextBasedLinuxInstall)

*	`dropbox.py` won't work without Python 2.6, but let's not risk messing up our official RHEL packages. Download ActivePython 2.6 (AS Package) from https://www.activestate.com/activepython/downloads [(ref)](http://stackoverflow.com/questions/1465036/install-python-2-6-in-centos)

*	As mentioned in the AP documentation [(ref)](http://docs.activestate.com/activepython/2.6/installnotes.html#aspackage), run the installer with `./install.sh` and install where desired. It defaults to */opt/ActivePython-2.6*, which is fine because it does not conflcit with the system default python 2.4 install. For my purposes, I had created a user called *dropbox* that did not have root privileges, so I installed to */home/dropbox/ActivePython-2.6*.

*	Edit *dropbox.py* and change `#!/usr/bin/python` to the path you just installed AP to. For my installation, it's set to `#!/home/dropbox/bin/ActivePython-2.6/bin/python`

*	Run `dropbox.py` without commands to see your available options.

Optional: To suppress LAN Sync broadcasts, download [dropboxp2p.py](http://wiki.dropbox.com/TipsAndTricks/TextBasedLinuxInstall?action=AttachFile&do=view&target=dropboxp2p.py) [(ref)](http://wiki.dropbox.com/TipsAndTricks/TextBasedLinuxInstall), edit the first line to use the ActivePython binary as you did for dropbox.py, and run `./dropboxp2p -d` to disable LAN syncing (unless you have other machines in the LAN with Dropbox installed, and your firewalls are set up appropriately).