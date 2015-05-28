---
layout: post
title: "All I want is PHP 5.2 on CentOS/RHEL!"
date: 2009-05-22 14:07
comments: true
categories:
---

*This post originally featured on the [Echo &amp; Co. blog](http://echo.co/blog/all-i-want-php-52-centosrhel)*.

**[An updated, easier method can be found here.](/blog/2011/01/25/easy-php-5-dot-2-rpms-on-centos/)**

At [EchoDitto](http://www.echoditto.com), most of our servers are running [CentOS Linux](http://www.centos.org/), which is a 100% binary-compatible version of the industry standard [Red Hat Enterprise Linux](http://www.redhat.com/rhel/server/) (RHEL) without any of the fees. The problem with RHEL/CentOS is that they shipped with PHP 5.1.6, and as of this writing PHP is at 5.2.9. That's not a big deal, being a minor point revision behind, until you come across an application or module that needs a minimum of version 5.2. The last thing I want to do is install packages from a third-party or build it from source and risk breaking other packages. So what's the answer? Building it from the source rpm. There's no better way to keep the system free of third-party packages but also up to date.

Why source RPMs and not the source from [PHP](http://www.php.net/downloads.php)? Building a source RPM is the same procedure used to build the packages that Red Hat creates, and includes the libraries and dependencies and sub-packages that you would normally get from Red Hat directly. It will also include all of the same build flags in your new version that were available in the older package, meaning your PHP scripts won't suddenly break. This ultimately ensures binary compatibility and the ability to easily upgrade if Red Hat or CentOS were to release a package newer than yours. 

Sure, you could enable the [CentOS Testing repository](http://wiki.centos.org/AdditionalResources/Repositories), but with all of the warnings about packages breaking your system, and configuring yum to only update PHP and not mess up mysql or Apache, I'd rather build and install only the packages that I need without worrying about breaking other packages. Want PHP 5.2 on your CentOS or RHEL server? Follow along.

* Start by downloading the latest PHP 5.2 source RPM from the aforementioned CentOS Testing repository from [http://dev.centos.org/centos/5/testing/SRPMS/](http://dev.centos.org/centos/5/testing/SRPMS/). As of this writing, they're at [php-5.2.6-2](http://dev.centos.org/centos/5/testing/SRPMS/php-5.2.6-2.el5s2.src.rpm). Why aren't we using the 5.1.6 source? The source RPM for 5.2.6 includes the latest patches from Red Hat that you can normally only get from being a part of the non-free Red Hat Network.

I'll assume you're running as root for this guide. Download the source RPM with wget:  

```bash
wget http://dev.centos.org/centos/5/testing/SRPMS/php-5.2.6-2.el5s2.src.rpm
```

* Install the source RPM

```bash
rpm -ivh php-5.2.6-2.el5s2.src.rpm
```

* If you get an error and the install fails, you may need to manually create the SOURCES directory and install again

```bash
mkdir -p /usr/src/redhat/SOURCES
rpm -ivh php-5.2.6-2.el5s2.src.rpm
```

* Download the latest stable release of PHP from [the php.net download page](http://www.php.net/downloads.php) and save it in the SOURCES folder. Change the version number to the latest version if needed.

```bash
cd /usr/src/redhat/SOURCES
wget http://us3.php.net/get/php-5.2.9.tar.gz/from/us.php.net/
```

* In the SPECS folder, copy the original php.spec file in case you make mistakes or wish to build 5.2.6.

```bash
cd /usr/src/redhat/SPECS
cp php.spec php526.spec
```

* Edit php.spec and change the version and revision numbers to be 5.2.9 (or later) and the revision to 1 since this is your first build.

<pre>Summary: The PHP HTML-embedded scripting language
Name: php
Version: 5.2.9
Release: 1%{?dist}
License: PHP
Group: Development/Languages
URL: http://www.php.net/</pre>

* Make sure you have the Development Tools group install for rpmbuild if you haven't already done so.

```bash
yum groupinstall "Development Tools"
```

* Try and build the spec file to see if you are missing any packages. For this tutorial, I am running from the minimal package set.

<pre>
rpmbuild -ba php.spec 
cat: /usr/include/httpd/.mmn: No such file or directory
error: Failed build dependencies:
	bzip2-devel is needed by php-5.2.9-1.i386
	curl-devel >= 7.9 is needed by php-5.2.9-1.i386
	db4-devel is needed by php-5.2.9-1.i386
	expat-devel is needed by php-5.2.9-1.i386
	gmp-devel is needed by php-5.2.9-1.i386
	aspell-devel >= 0.50.0 is needed by php-5.2.9-1.i386
	httpd-devel >= 2.0.46-1 is needed by php-5.2.9-1.i386
	libjpeg-devel is needed by php-5.2.9-1.i386
	libpng-devel is needed by php-5.2.9-1.i386
	pam-devel is needed by php-5.2.9-1.i386
	openssl-devel is needed by php-5.2.9-1.i386
	sqlite-devel >= 3.0.0 is needed by php-5.2.9-1.i386
	zlib-devel is needed by php-5.2.9-1.i386
	pcre-devel >= 6.6 is needed by php-5.2.9-1.i386
	readline-devel is needed by php-5.2.9-1.i386
	krb5-devel is needed by php-5.2.9-1.i386
	libc-client-devel is needed by php-5.2.9-1.i386
	cyrus-sasl-devel is needed by php-5.2.9-1.i386
	openldap-devel is needed by php-5.2.9-1.i386
	mysql-devel >= 4.1.0 is needed by php-5.2.9-1.i386
	postgresql-devel is needed by php-5.2.9-1.i386
	unixODBC-devel is needed by php-5.2.9-1.i386
	libxml2-devel is needed by php-5.2.9-1.i386
	net-snmp-devel is needed by php-5.2.9-1.i386
	libxslt-devel >= 1.0.18-1 is needed by php-5.2.9-1.i386
	libxml2-devel >= 2.4.14-1 is needed by php-5.2.9-1.i386
	ncurses-devel is needed by php-5.2.9-1.i386
	gd-devel is needed by php-5.2.9-1.i386
	freetype-devel is needed by php-5.2.9-1.i386
</pre>

* Install any required packages with yum. If any of these packages have other dependencies they will be downloaded as well. This might take a while.

```bash
yum install bzip2-devel curl-devel db4-devel expat-devel gmp-devel aspell-devel httpd-devel libjpeg-devel libpng-devel pam-devel openssl-devel sqlite-devel zlib-devel pcre-devel readline-devel krb5-devel libc-client-devel cyrus-sasl-devel openldap-devel mysql-devel postgresql-devel unixODBC-devel libxml2-devel net-snmp-devel libxslt-devel libxml2-devel ncurses-devel gd-devel freetype-devel
```

* Try and build the spec file again to see if any patches fail to load properly. This usually happens if a patch has been rolled into the mainline PHP project and is no longer needed.

<pre>
rpmbuild -ba php.spec
Executing(%prep): /bin/sh -e /var/tmp/rpm-tmp.81391
+ umask 022
+ cd /usr/src/redhat/BUILD
+ LANG=C
+ export LANG
+ unset DISPLAY
+ cd /usr/src/redhat/BUILD
+ rm -rf php-5.2.9
+ /bin/gzip -dc /usr/src/redhat/SOURCES/php-5.2.9.tar.gz
+ tar -xf -
+ STATUS=0
... output snipped ...
Patch #32 (php-5.2.5-systzdata.patch):
+ patch -p1 -b --suffix .systzdata -s
+ echo 'Patch #50 (php-5.2.4-tests-dashn.patch):'
Patch #50 (php-5.2.4-tests-dashn.patch):
+ patch -p1 -b --suffix .tests-dashn -s
Reversed (or previously applied) patch detected!  Assume -R? [n] 
Apply anyway? [n] 
1 out of 1 hunk ignored -- saving rejects to file ext/standard/tests/file/bug26615.phpt.rej
1 out of 1 hunk FAILED -- saving rejects to file ext/standard/tests/file/proc_open01.phpt.rej
Reversed (or previously applied) patch detected!  Assume -R? [n] 
Apply anyway? [n] 
1 out of 1 hunk ignored -- saving rejects to file ext/standard/tests/file/bug26938.phpt.rej
error: Bad exit status from /var/tmp/rpm-tmp.81391 (%prep)


RPM build errors:
    Bad exit status from /var/tmp/rpm-tmp.81391 (%prep)
</pre>

* Looks like Patch 50 failed. Edit the spec file and comment out the line that begins with %patch50

<pre>
%patch30 -p1 -b .dlopen
%patch31 -p1 -b .easter
%patch32 -p1 -b .systzdata

#%patch50 -p1 -b .tests-dashn
%patch51 -p1 -b .tests-wddx
</pre>

* For 5.2.9 as of this writing, this is the only patch that fails to load properly. In later versions, you may find that other patches fail to load, and you should comment those out as well.

* Build the spec file again and after several minutes it will be complete and you'll have a handful of rpms in /usr/src/redhat/RPMS/i386

<pre>
rpmbuild -ba php.spec
Executing(%prep): /bin/sh -e /var/tmp/rpm-tmp.77534
+ umask 022
+ cd /usr/src/redhat/BUILD
+ LANG=C
+ export LANG
+ unset DISPLAY
+ cd /usr/src/redhat/BUILD
+ rm -rf php-5.2.9
+ /bin/gzip -dc /usr/src/redhat/SOURCES/php-5.2.9.tar.gz
+ tar -xf -
+ STATUS=0
... output snipped ...
Wrote: /usr/src/redhat/RPMS/i386/php-ncurses-5.2.9-1.i386.rpm
Wrote: /usr/src/redhat/RPMS/i386/php-gd-5.2.9-1.i386.rpm
Wrote: /usr/src/redhat/RPMS/i386/php-bcmath-5.2.9-1.i386.rpm
Wrote: /usr/src/redhat/RPMS/i386/php-dba-5.2.9-1.i386.rpm
Wrote: /usr/src/redhat/RPMS/i386/php-debuginfo-5.2.9-1.i386.rpm
Executing(%clean): /bin/sh -e /var/tmp/rpm-tmp.7525
+ umask 022
+ cd /usr/src/redhat/BUILD
+ cd php-5.2.9
+ '[' /var/tmp/php-5.2.9-1-root-root '!=' / ']'
+ rm -rf /var/tmp/php-5.2.9-1-root-root
+ rm files.bcmath files.common files.dba files.dbase files.dom files.gd files.imap [snip]
+ exit 0
</pre>

* You'll probably want to also build the php-extras source RPM as well. php-extras provides extensions for dbase, readline, mcrypt, mhash, tidy and mssql. Since there is no package in CentOS Testing, grab the latest from 5.1.6.

```bash
wget http://mirror.centos.org/centos/5/extras/SRPMS/php-extras-5.1.6-15.el5.centos.1.src.rpm
```

* And install it.

```bash
rpm -ivh php-extras-5.1.6-15.el5.centos.1.src.rpm
```

* Copy the php-extras.spec file like you did for php.spec so you can have the original.

```bash
cd /usr/src/redhat/SPECS
cp php-extras.spec php-extras516.spec
```

* We ultimately need to replace the patches in php-extras.spec with the same from the much newer php.spec, as well as the ABI version numbers in the beginning of the file. [Webtatic.com has already done this](http://www.webtatic.com/blog/2009/05/installing-php-526-extra-extensions/) and you can go ahead and [download the edited spec file](http://www.webtatic.com/files/2009/05/php-extras.spec), which also includes new PHP 5.2 extensions for json and filter. Remember, if you had a failed patch from php.spec, you'll need to comment it out in php-extras.spec as well. For 5.2.9, I had to comment out patch50 again.

<pre>
%patch32 -p1 -b .systzdata

%patch50 -p1 -b .tests-dashn
%patch51 -p1 -b .tests-wddx

%build
</pre>

* Once your php-extras.spec file is up to date, build the spec file and see if you're missing any dependencies.

<pre>
rpmbuild -ba php-extras.spec 
error: Failed build dependencies:
	php-devel = 5.2.9 is needed by php-extras-5.2.9-1.i386
	libmcrypt-devel is needed by php-extras-5.2.9-1.i386
	mhash-devel is needed by php-extras-5.2.9-1.i386
	libtidy-devel is needed by php-extras-5.2.9-1.i386
	freetds-devel is needed by php-extras-5.2.9-1.i386
</pre>

* Let's go back and install the php package that we need. If you only want the minimum needed to build php-extras, start by installing php-devel and seeing what it asks for, and install it's dependencies.

<pre>
cd /usr/src/redhat/RPMS/i386
rpm -ivh php-devel-5.2.9-1.i386.rpm 
error: Failed dependencies:
	php = 5.2.9-1 is needed by php-devel-5.2.9-1.i386
rpm -ivh php-5.2.9-1.i386.rpm php-devel-5.2.9-1.i386.rpm 
error: Failed dependencies:
	php-cli = 5.2.9-1 is needed by php-5.2.9-1.i386
	php-common = 5.2.9-1 is needed by php-5.2.9-1.i386
rpm -ivh php-5.2.9-1.i386.rpm php-devel-5.2.9-1.i386.rpm php-cli-5.2.9-1.i386.rpm php-common-5.2.9-1.i386.rpm 
Preparing...                ########################################### [100%]
   1:php-common             ########################################### [ 25%]
   2:php-cli                ########################################### [ 50%]
   3:php                    ########################################### [ 75%]
   4:php-devel              ########################################### [100%]
</pre>

* After installing those four packages, install the other packages with yum.

```bash
yum install libmcrypt-devel mhash-devel libtidy-devel freetds-devel
```

* Now that all of the dependencies are resolved, build the php-extras.spec file.

```bash
cd /usr/src/redhat/SPECS
rpmbuild -ba php-extras.spec
```

* If you're using RHEL, the build might fail for two reasons: a library for tidy is needed, and mhash-devel isn't available for RHEL. Both are easy to fix. Install 'tidy' with yum, and while mhash-devel isn't available, you can download libmhash-devel and edit the spec file to require libmhash-devel instead of mhash-devel.

You should now have all of your RPMs in /usr/src/redhat/RPMS/i386 (or x86_64) and install whichever packages you want. If in doubt, install them all. They're mainly static libraries and won't bloat your system. You may not want to install the debuginfo packages since you may find little use for them.

Staying updated is easy, download the source from php.net into /usr/src/redhat/SOURCES and edit the spec files for the new version numbers, rebuild, and install RPMs. If you're concerned about new patches being needed, you can grab the latest source RPM from Centos: [http://dev.centos.org/centos/5/testing/SRPMS/](http://dev.centos.org/centos/5/testing/SRPMS/). If CentOS were to ever come out with a newer version, or you want to subscribe to a yum repo and they provided a higher version, it would update as normal. You would need to rebuild in pear/pecl libraries by uninstall+installing them.

Yes, this is an involved process the first time around, but after doing this once you only need to download the latest source from php.net and edit the spec file version number and rebuild and update. If an old patch fails in a newer version you'll need to comment it out, but it's that simple. 

This also works for 64-bit installations. Thanks to [DiNo](http://www.atoomnet.net/centos_updated_php.php) for helping me get PHP built on a 64-bit RHEL installation.

If you'd like to skip the steps and get right to building, you can download and install the source RPMs and build again the spec files. If you'd like to do itself, you can compare my spec files to yours. Enjoy the latest PHP hotness! 

* [PHP Spec files](http://www.filedropper.com/php-spec-files)
* [php 5.2.9 source RPM](http://www.filedropper.com/php-529-1src) MD5 = 008d5f2ced015c5f6463afb616737b04
* [php-extras 5.2.9 source RPM](http://www.filedropper.com/php-extras-529-1src) MD5 = 2f8a7e815170bf31509180c196ade19a