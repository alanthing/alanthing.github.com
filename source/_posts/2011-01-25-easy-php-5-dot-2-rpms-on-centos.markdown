---
layout: post
title: "Easy PHP 5.2 RPMs on CentOS"
date: 2011-01-25 10:13
comments: true
categories: 
---

*This post originally featured on the [Echo &amp; Co. blog](http://echo.co/blog/easy-php-52-rpms-centos)*.

I had previously [written a post](http://echodittolabs.org/blog/2009/05/all-i-want-php-52-centosrhel) on one method of upgrading PHP from 5.1 to 5.2 on CentOS and Red Hat servers by creating new RPMs. Since then, I have found a much better way to create PHP 5.2.17 (or newer) RPMs to easily upgrade (and later remove, if you want) the older version available by default. I'll presume you have no prior experience building PHP RPMs.

We start by adding the [Extra Packages for Enterprise Linux (EPEL) repository](http://fedoraproject.org/wiki/EPEL) to our server. EPEL is safe to add as an additional repository and leave enabled because it [provides complementary packages only](http://fedoraproject.org/wiki/EPEL/FAQ#How_is_EPEL_different_from_other_third_party_repositories_for_RHEL_and_derivatives.3F) and will not conflict with the CentOS or Red Hat repositories. Since their installation method may change, I won't reproduce it here; instead head over to the [installation portion of the FAQ](http://fedoraproject.org/wiki/EPEL/FAQ#howtouse) and it's just one line to get started with EPEL.

With EPEL enabled, we will need just one new package to build RPMs: [Mock](http://fedoraproject.org/wiki/Projects/Mock). Mock will create a chroot, download what's necessary to build your RPM, build, and cleanup afterwards. Mock only requires a couple of python packages as dependencies; it will not require a compiler or anything like that since mock will grab them as needed during operation. I will presume you are not running as root so I will use sudo as necessary. To install mock:

```bash
sudo yum install mock
```

That's it. You can run all mock operations with the 'sudo' command to run as root, but your packages will be built inside the chroot as a non-root user. Or, if you prefer to not use sudo when running mock, add your user to the mock group (where username is your user):

```bash
sudo /usr/sbin/usermod -G mock username
```

Using the source code from php.net and creating a RPM spec file from scratch would be a lot of work, so we'll grab a source RPM for PHP 5.2. I've found that [Remi](http://blog.famillecollet.com/) has the most up-to-date RPM spec with the latest patches and robust install scripts. [Browse Remi's SRPMS](http://rpms.famillecollet.com/SRPMS/) and download the latest PHP 5.2 file. As of this writing, it's php-5.2.17-1.remi.src.rpm. On your system, download it with wget in your home directory:

```bash
cd ~
wget http://rpms.famillecollet.com/SRPMS/php-5.2.17-1.remi.src.rpm
```

Now, we could just run mock and rebuild this source RPM and hopefully install the results, but it would fail due to not having sqlite2-devel installed. Remi provides sqlite2 and sqlite2-devel, so you could add Remi's yum repository to your mock settings, but then it would grab Remi's other updated packages like MySQL 5.5 and others during the PHP build. My goal here is to only build PHP against Base CentOS and EPEL packages. So, we have to edit the spec file and rebuild the source RPM before passing it off to mock. Don't worry; while there are a few steps, it's not very difficult if you follow closely.

To unpack the source RPM we just downloaded into our home directory, we'll need to set up the .rpmmacros file in our home directory. Why? Without it, it will attempt to extract to /usr/src/redhat, and only root can use that directory, and you should [never build any packages as a privileged user](http://wiki.centos.org/HowTos/SetupRpmBuildEnvironment). We'll create the macros file and the directories it needs from your home directory:

```bash
cd ~
echo '%_topdir %(echo $HOME)/rpmbuild' > ~/.rpmmacros
mkdir -p ~/rpmbuild/{SOURCES,SPECS,SRPMS}
```

Unpack the source RPM. This will extract the PHP 5.2.17 source and any patches and configuration files into ~/rpmbuild/SOURCES, and the spec file will be in ~/rpmbuild/SPECS:

```bash
rpm -ivh php-5.2.17-1.remi.src.rpm
```

Now we need to make a couple of changes to the spec file. We'll change to the SPECS directory and make a copy of Remi's spec file so we can keep it for reference:

```bash
cd ~/rpmbuild/SPECS
cp -a php52.spec php.spec
```

Open up php.spec in your favorite text editor and let's have the PHP configure script use the bundled SQLite2 libraries instead of searching for ones on your system:

1. Find the line "BuildRequires: sqlite2-devel >= 2.8.0" and delete it
2. Find "--with-sqlite=shared,%{_prefix} \" and replace it with "--with-sqlite=shared \"
3. Optionally: Add your own lines to the %changelog describing that you removed the sqlite2-devel build requirement
4. Optionally: Remove the first 4 lines after "%pre common" to remove Remi's note about their forums

It took 2 small tweaks to remove the sqlite2-devel RPM requirement not found in our standard repositories. See [php.spec.diff](http://paste.ly/4Yrx) to see only the changes we made; or [php.spec](http://paste.ly/4Ys5) for the complete file if you do not want to edit it yourself.

The hard part is now over! It's easy from here on out. First step is to build the newly-edited spec into your own source RPM. But first, if you don't have "rpmbuild" available, it's a quick install via yum:

```bash
sudo yum install rpm-build
```

It's possible your system may have already had the yum package rpm-build installed. We'll build our source RPM with "-bs" to **b**uild the **s**ource, and "--nodeps" to prevent our system from checking all of the "Requires" in the spec since we're just building the source:

```
rpmbuild --nodeps -bs php.spec
Wrote: /home/username/rpmbuild/SRPMS/php-5.2.17-1.src.rpm
```

rpmbuild will show you where it wrote your source RPM. All we have to do is provide it to mock; "-v" will give us verbose output and "-r epel-5-x86_64" will use the mock profile at "/etc/mock/epel-5-x86_64.cfg". If you're on a 32-bit system, change "epel-5-x86_64" to "epel-5-i386":

```bash
cd ~/rpmbuild/SRPMS
mock -v -r epel-5-x86_64 php-5.2.17-1.src.rpm
```

The output shows you that mock is creating a chroot, downloading the bare essentials needed for a chroot and compiling tools, ccache to speed up compilation, and the packages in all "Requires" lines in the spec file. When mock is finished with the build after several minutes, you'll see output similar to:

```
INFO: Done(php-5.2.17-1.src.rpm) Config(epel-5-x86_64) 10 minutes 20 seconds
INFO: Results and/or logs in: /var/lib/mock/epel-5-x86_64/result
```

The /var/lib/mock/epel-5-x86_64/results/ folder will contain all of your PHP RPMs, including another source RPM. You can now either install these PHP packages by hand, or you can dump them into [your own Yum repository](http://www.techrepublic.com/blog/opensource/create-your-own-yum-repository/609) and install with yum.

### TL;DR

```bash
sudo yum install mock
sudo /usr/sbin/usermod -G mock username
cd ~
wget http://rpms.famillecollet.com/SRPMS/php-5.2.17-1.remi.src.rpm
echo '%_topdir %(echo $HOME)/rpmbuild' > ~/.rpmmacros
mkdir -p ~/rpmbuild/{SOURCES,SPECS,SRPMS}
rpm -ivh php-5.2.17-1.remi.src.rpm
cd ~/rpmbuild/SPECS
cp -a php52.spec php.spec
sed -i "/BuildRequires\: sqlite2-devel/d" php.spec
sed -i "s/--with-sqlite=shared,%{_prefix}/--with-sqlite=shared/" php.spec
sudo yum install rpm-build
rpmbuild --nodeps -bs php.spec
cd ~/rpmbuild/SRPMS
mock -v -r epel-5-x86_64 php-5.2.17-1.src.rpm
```

Resulting RPMs will be in /var/lib/epel-5-x86_64/results. If on a 32-bit system, replace epel-5-x86_64 with epel-5-i386.