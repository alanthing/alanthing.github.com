---
layout: post
title: "Fix pear permissions problem on Mac OS X"
date: 2011-02-15 16:34
comments: true
categories:
---

On Snow Leopard, you can install drush without using sudo. A dependency for installing drush is downloading a Pear library. The following example should allow you to be able to use the following commands to install drush to /usr/local/drush with a symlink in /usr/local/bin/drush, but it fails on the 'pear download' step, even though it's writing to a user-writeable directory. Read on the figure out how to fix this annoying problem.

```
svn co http://subversible.svn.beanstalkapp.com/modules/drush/tags/DRUPAL-7--4-2/ /usr/local/drush 
cd /tmp 
pear download Console_Table 
tar zxvf `ls Console_Table-*.tgz` `ls Console_Table-*.tgz | sed -e 's/\.[a-z]\{3\}$//'`/Table.php 
mv `ls Console_Table-*.tgz | sed -e 's/\.[a-z]\{3\}$//'`/Table.php 
/usr/local/drush/includes/table.inc $ rm -fr Console_Table-* 
ln -s /usr/local/drush/drush /usr/local/bin/drush
```

This assumes 1) /usr/local exists and you can write to it, and 2) you've added the following to your ~/.profile (or ~/.bashrc or ~/.bash_profile):

```
# Use the built-in PEAR and PECL scripts
alias pear="/usr/bin/php /usr/lib/php/pearcmd.php"
alias pecl="/usr/bin/php /usr/lib/php/peclcmd.php"
```

But, OS X has a strange permissions problem when using pear for the first time. Running 'pear download' should not require sudo, but if you've never used pear before you'll get this error:

```
$ pear download Console_Table 

Warning: touch(): Unable to create file /usr/lib/php/.lock because Permission denied in PEAR/Registry.php on line 835 

Warning: touch(): Unable to create file /usr/lib/php/.lock because Permission denied in /usr/lib/php/PEAR/Registry.php on line 835 
could not create lock file: fopen(/usr/lib/php/.lock): failed to open stream: No such file or directory 
invalid package name/package file "Console_Table" 
download failed
```

Strange, right? Administrators in OS X are in the wheel group, but /usr/lib/php is not writable by the group. Before you go changing permissions (which might not be retained after an OS X update), all you have to do is run something as root with pear:

```
$ sudo pear list
Installed packages, channel pear.php.net:
=========================================
Package          Version State
Archive_Tar      1.3.3   stable
Console_Getopt   1.2.3   stable
PEAR             1.8.0   stable
Structures_Graph 1.0.2   stable
XML_Util         1.2.1   stable
```

The permissions and ownership of /usr/lib/php are still the same, but there is now the file /usr/lib/php/.lock

```
$ ls -Al /usr/lib/php/.lock
-rw-r--r--  1 root  wheel  0 Feb 15 16:08 /usr/lib/php/.lock
```

You'd think that when pear has completed running the list command it would delete the .lock file, but it doesn't. But since it's there, the original pear download command will now run without sudo:

```
$ pear download Console_Table
downloading Console_Table-1.1.4.tgz ...
Starting to download Console_Table-1.1.4.tgz (9,369 bytes)
.....done: 9,369 bytes
File /private/tmp/Console_Table-1.1.4.tgz downloaded
```

Of course, running 'pear install' or many other commands will require sudo, but it's silly that commands that do not write any files to /usr/lib/php or its children would require sudo.

Unrelated to the pear permissions issue, but related to my drush install script; this assumes that /usr/local and /usr/local/bin exist. By default, /usr/local/bin is in the $PATH regardless of whether or not the folder exists, but on a vanilla OS X install /usr/local does not exist. I have both Xcode and homebrew installed. I know homebrew will create /usr/local for you, but I'm not sure about Xcode. If you don't want to have either installed but want drush installed to /usr/local, run this:

```
sudo mkdir -p /usr/local/bin
sudo chown -R root:wheel /usr/local
```