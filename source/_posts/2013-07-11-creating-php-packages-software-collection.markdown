---
layout: post
title: "Creating PHP packages as a Software Collection"
date: 2013-07-11 22:33
comments: true
categories: 
---

Maintaining multiple versions of software can be tricky. Most of the explanations I have come across suggest using the version provided by your Linux distribution and installing others from source using a custom path, like `./configure --prefix=/opt/php-5.3.26; make; make install`, but it's not pleasant to update, manage small sub-packages, or make init.d scripts. The ideal option is to use packages that install to another location alongside the defaults, but hacking up spec files to override the default RPM macros like `%configure` (for CentOS and RHEL anyway) is not fun. This is where [Red Hat Software Collections](https://access.redhat.com/site/documentation/en-US/Red_Hat_Developer_Toolset/1/html/Software_Collections_Guide/) come in to play. While Red Hat recently announced the [Red Hat Software Collections 1.0 Beta](http://www.redhat.com/about/news/archive/2013/6/red-hat-software-collections-1.0-beta-now-available) for alternate versions of PHP, Perl, MySQL, and more, let's step through how to create your own Software Collection, or SCL. I'll show how to create PHP 5.3.26 to be installed in /opt/rh/ by converting existing SPEC files, and I'll also include diff files for how I built PHP 5.2.17 as an SCL.

If you're just here to get RPMs and don't much care for how they were made, or if you'd like to get the source RPMs and learn for yourself, you can visit our repository and get whatever you'd like, including Yum repos:

* [http://yum.echoditto.com/php53/](http://yum.echoditto.com/php53/)
* [http://yum.echoditto.com/php52/](http://yum.echoditto.com/php52/)

Before I get too deep into the instructions, I'd like to share the resources I used:

*   [Red Hat's documentation on converting SPEC files to SCLs](https://access.redhat.com/site/documentation/en-US/Red_Hat_Developer_Toolset/1/html/Software_Collections_Guide/sect-Converting_a_Conventional_Spec_File.html)
*   [PHP 5.4 SCL SRPMs](http://people.redhat.com/rcollet/php54/rhel/5/SRPMS/) from [Remi Collet](http://blog.famillecollet.com/post/2012/11/20/PHP-5.4-as-Software-Collection)
*   [PHP 5.3 SRPMs from AtomiCorp](http://www2.atomicorp.com/channels/source/php/), because I wanted [mysqlnd](http://php.net/manual/en/book.mysqlnd.php) and didn't want to hack up the Red Hat SPEC to add it
*   [PHP 5.2 SRPMs from alt.ru](http://centos.alt.ru/pub/repository/centos/5/SRPMS/), which contains many backported patches and PHP-FPM

An important point to be learned is: **Software Collections are essentially easy-to-use RPM macros** that make installing packages into non-standard directories a piece of cake. As I explain various steps, I'll attempt to explain how the Software Collections function and how I've seen others set them up so we're following best practices in ours.

## Explaining the Software Collection base

Every Software Collection needs a base package that sets up the directory structure for the packages. When you use Red Hat's method of building SCLs, specifically via the *scl-utils-build* package, the base package will create **/opt/rh/%{scl_prefix}/root** with subdirectories like bin/, usr/, etc/, as so on.

For the SCL named *php53*, here is the base directory structure:

<pre>
/opt/rh/php53/root/bin
/opt/rh/php53/root/boot
/opt/rh/php53/root/dev
/opt/rh/php53/root/etc
/opt/rh/php53/root/home
/opt/rh/php53/root/lib
/opt/rh/php53/root/lib64
/opt/rh/php53/root/media
/opt/rh/php53/root/mnt
/opt/rh/php53/root/opt
/opt/rh/php53/root/proc
/opt/rh/php53/root/root
/opt/rh/php53/root/sbin
/opt/rh/php53/root/selinux
/opt/rh/php53/root/srv
/opt/rh/php53/root/sys
/opt/rh/php53/root/tmp
/opt/rh/php53/root/usr
/opt/rh/php53/root/var
</pre>

Most of the SCLs I have seen contain an `enable` script at **/opt/rh/%{scl_prefix}/enable** that will add the binaries, man pages, and library paths to the session. In this case, running `source /opt/rh/php53/enable` will run the following:

```bash
export PATH=/opt/rh/php52/root/usr/bin:/opt/rh/php52/root/usr/sbin:$PATH
export LD_LIBRARY_PATH=/opt/rh/php52/root/usr/lib64:$LD_LIBRARY_PATH
export MANPATH=/opt/rh/php52/root/usr/share/man:$MANPATH
```

The base package will also create a *%{scl_prefix}-build* subpackage that creates RPM macros that will come in handy as the SPEC file is edited.

## Building the Software Collection base

*   Start with [http://people.redhat.com/rcollet/php54/rhel/5/SRPMS/php54-1-2.el5.src.rpm](http://people.redhat.com/rcollet/php54/rhel/5/SRPMS/php54-1-2.el5.src.rpm) because it's ready for CentOS 5 and 6

*   Rename or copy php54.spec to php53.spec

*   Set the global variable `scl` to be **php53**:

        %global scl php53

*   If you haven't already, install *rpm-build* and *mock*, and you'll also need *scl-utils-build* for later:

        yum -y install rpm-build mock scl-utils-build

*   CentOS 5 needs additional rpmbuild options when using a CentOS 6 build host because [CentOS 6 uses stronger hashes](https://fedoraproject.org/wiki/Features/StrongerHashes) by default

    *   CentOS 5: 
    
            rpmbuild --define "_source_filedigest_algorithm md5" --define "_binary_filedigest_algorithm md5" -bs --nodeps php53.spec

    *   CentOS 6: 

            rpmbuild -bs --nodeps php53.spec

*   Build with mock (repeat el6 instructions for el5):

        mock -v -r epel-6-x86_64 php53-1-2.el5.src.rpm

*   Copy resulting files to local folder and create a local Yum repo to be used later (repeat el6 instructions for el5):

        mkdir -vp ~/rpmbuild/RPMS/repo/el6/{x86_64,SRPMS}
        cp -va /var/lib/mock/epel-6-x86_64/result/*.{noarch,x86_64}.rpm ~/rpmbuild/RPMS/repo/el6/x86_64
        cp -va /var/lib/mock/epel-6-x86_64/result/*.src.rpm ~/rpmbuild/RPMS/repo/el6/SRPMS
        createrepo -v -d ~/rpmbuild/RPMS/repo/el6/x86_64

## Mock

We used `mock` to build the SCL base package, and to make our lives significantly easier, we'll continue to use it with some modifications specific to our SCL.

For all remaining steps, unless there are specific notes for CentOS 5 and 6, repeat the 6/el6 instructions for 5/el5.

*   Change to the mock configuration directory:

        cd /etc/mock

*   Copy the default EPEL files to new ones specific to our SCL:

        cp -a epel-6-x86_64.cfg epel-6-x86_64-scl-php53.cfg

*   Edit *epel-6-x86_64-scl-php53.cfg* and:

    *   Add **-scl-php53** to the existing value of *config_opts['root']*:

            config_opts['root'] = 'epel-6-x86_64-scl-php53'

    *   CentOS 5 and 6 have different options for *config_opts['chroot_setup_cmd']*:

        *   CentOS 5: Add **scl-utils-build php53-build** the following to the existing value:

                config_opts['chroot_setup_cmd'] = 'install buildsys-build scl-utils-build php53-build'

        *   CentOS 6: Change from 'groupinstall buildsys-build' to **install @buildsys-build scl-utils-build php53-build** (the @ will grab the group and then still allow for non-group packages to be added):

                config_opts['chroot_setup_cmd'] = 'install @buildsys-build scl-utils-build php53-build'

*   Add the local repo where you left the files from the section above as a new repo; I added mine above *[base]*. **Change /home/alan to the appropriate path for your system!**

        [localrepo-6-x86_64]
        name=localrepo-6-x86_64
        enabled=1
        baseurl=file:///home/alan/rpmbuild/RPMS/repo/el6/x86_64

## PHP

*   Use latest php-5.3.x SRPM from [http://www2.atomicorp.com/channels/source/php/](http://www2.atomicorp.com/channels/source/php/)

*   Install the SRPM

        rpm -ivh php-5.3.26-19.art.src.rpm

*   Go to the SPECS folder, typically at *~/rpmbuild/SPECS*, and copy *php-art.spec* to *php.spec*

        cp -v php-art.spec php.spec

*   Several changes are needed to add the SCL macros to the SPEC file; see the [Red Hat documentation](https://access.redhat.com/site/documentation/en-US/Red_Hat_Developer_Toolset/1/html/Software_Collections_Guide/sect-Converting_a_Conventional_Spec_File.html). Download my patch file for a full list of changes needed to convert AtomiCorp's PHP SPEC to one that can be used with and without SCL options: [php_scl.diff](/files/php_scl.diff)

        curl --remote-name --location http://alanthing.com/files/php_scl.diff
Let's step through the changes:

    *   The first section allows for variables beginning with `_root_` to be available if it's not being built with a SCL. As I said earlier, I heavily borrowed from Remi's PHP 5.4 SCL SRPM. The `_root_` macros are provided by the *scl-utils-build* package, but this block allows them to work if an SCL is not being used.

          +%if 0%{?scl:1}
          +%scl_package php
          +%else
          +%global pkg_name          %{name}
          +%global _root_sysconfdir  %{_sysconfdir}
          +%global _root_bindir      %{_bindir}
          +%global _root_sbindir     %{_sbindir}
          +%global _root_includedir  %{_includedir}
          +%global _root_libdir      %{_libdir}
          +%global _root_prefix      %{_prefix}
          +%if 0%{?rhel} < 6
          +%global _root_initddir    /etc
          +%else
          +%global _root_initddir    %{_initddir}
          +%endif
          +%endif

    *   These values allow you to use the SCL mod_php with the built-in version of Apache

          +%global _httpd_confdir     %{_root_sysconfdir}/httpd/conf.d
          +%global _httpd_moddir      %{_libdir}/httpd/modules
          +%global _root_httpd_moddir %{_root_libdir}/httpd/modules

    *   There are other changes scattered throughout the file to force using locations outside of the SCL, like:

          -%global httpd_mmn %(cat %{_includedir}/httpd/.mmn || echo missing-httpd-devel)
          +%global httpd_mmn %(cat %{_root_includedir}/httpd/.mmn || echo missing-httpd-devel)

    *   In all of the *Name:* lines, add the macro **%{?scl_prefix}**, which will only be used when building an SCL. For example, if you use the SCL rpmbuild options, the common package here would be called *php53-php-common*, but without it would still be called *php-common*. This applies for the main *Name* and sub-packages. It's also used for *Obsoletes*, *Conflicts*, *Provides*, and *Requires*.
    
          -Name: %{phpname}
          +Name: %{?scl_prefix}%{phpname}
          
          -Obsoletes: php-mhash
          -Conflicts: php-zend-optimizer
          +Obsoletes: %{?scl_prefix}php-mhash
          +Conflicts: %{?scl_prefix}php-zend-optimizer

    *   The main package would provide *mod_php* for the built-in Apache, so this addition prevents a collision of having both `php` and `php53-php` installed:

          +%if 0%{?scl:1}
          +Conflicts: php53, php
          +%else
           # php53
           Obsoletes: php53
          +%endif

    *   As per the Red Hat documentation, a main package (not php, but php-common, since php-common is needed by every php package) must require the base package

          %{?scl:Requires:%scl_runtime}

    *   The AtomiCorp SPEC is carefully designed to prevent conflicts with the CentOS 5 packages called *php53* that are not part of an SCL. I added a change to only keep the *Obsoletes* if it's not an SCL

          +%if 0%{?!scl:1}
           Obsoletes: php53-imap
          +%endif

    *   I made a few improvements that aren't related to SCL. I won't list them all here, but one that I noticed was that libevent is no longer needed after PHP 5.3.4, as my comment suggests explaining why I comment it out:

          +# libevent needed *until* 5.3.4, see http://bugzilla.redhat.com/show_bug.cgi?id=835671
          +#BuildRequires: libevent-devel >= 1.4.11

    *   A lot of the compile options will use the built-in libraries, like:

          -	--with-freetype-dir=%{_prefix} \
          -	--with-png-dir=%{_prefix} \
          -	--with-xpm-dir=%{_prefix} \
          + --with-freetype-dir=%{_root_prefix} \
          + --with-png-dir=%{_root_prefix} \
          + --with-xpm-dir=%{_root_prefix} \

    *   In a few places I use `sed` to fix config files, here's one where I ensure the SCL root is used for the *session.save_path* by changing */var/lib* to *%{_localstatedir}/lib*

          -%{__sed} -e '/session.save_path/s/php/%{phpname}/' %{SOURCE2} >$RPM_BUILD_ROOT%{_sysconfdir}/php.ini
          +%{__sed} -e 's|/var/lib/php/session|%{_localstatedir}/lib/%{phpname}/session|' %{SOURCE2} >$RPM_BUILD_ROOT%{_sysconfdir}/php.ini

    *   Instead of installing the Apache module files in the system default directories, I decided to keep them in the SCL root and create symbolic links, though I do put the mod_php Apache configuration file in /etc/httpd/conf.d

          -  >$RPM_BUILD_ROOT/%{_origsysconfdir}/httpd/conf.d/%{phpname}.conf
          +  >$RPM_BUILD_ROOT/%{_httpd_confdir}/%{?scl_prefix}%{phpname}.conf
          +%endif
          +
          +# Symlink SCL DSOs into real httpd moddir, mod_php config into real httpd config directory
          +%if %{?scl:1}0
          +install -m 755 -d $RPM_BUILD_ROOT%{_root_httpd_moddir}
          +install -m 755 -d $RPM_BUILD_ROOT%{_httpd_confdir}
          +%if %{phpname} == php
          +ln -s %{_httpd_moddir}/libphp5.so $RPM_BUILD_ROOT%{_root_httpd_moddir}/libphp5.so
          +ln -s %{_httpd_moddir}/libphp5-zts.so $RPM_BUILD_ROOT%{_root_httpd_moddir}/libphp5-zts.so
          +%else
          +ln -s %{_httpd_moddir}/libphp5.so $RPM_BUILD_ROOT%{_root_httpd_moddir}/lib%{phpname}.so
          +ln -s %{_httpd_moddir}/libphp5-zts.so $RPM_BUILD_ROOT%{_root_httpd_moddir}/lib%{phpname}-zts.so
          +%endif
           %endif

    *   A common practice I observed in other SCL files was putting the init.d script in */etc/init.d* instead of the SCL root with prepending the SCL name. For example, if you build with SCL options, the PHP-FPM script would be at */etc/init.d/php53-php-fpm*, or without it would be */etc/init.d/php-fpm*, either one using the correct paths:

          -install -m 755 -d $RPM_BUILD_ROOT%{_originitdir}
          -install -m 755 %{SOURCE6} $RPM_BUILD_ROOT%{_originitdir}/php-fpm
          +install -m 755 -d $RPM_BUILD_ROOT%{_root_initddir}
          +install -m 755 %{SOURCE6} $RPM_BUILD_ROOT%{_root_initddir}/%{?scl_prefix}%{phpname}-fpm
          +sed -e '/daemon/s:php-fpm:/usr/sbin/php-fpm:' \
          +    -i $RPM_BUILD_ROOT%{_root_initddir}/%{?scl_prefix}%{phpname}-fpm
          +sed -e '/php-fpm.pid/s:/var:%{_localstatedir}:' \
          +    -e '/subsys/s/php-fpm/%{?scl_prefix}%{phpname}-fpm/' \
          +    -e 's:/etc/sysconfig/php-fpm:%{_sysconfdir}/sysconfig/%{phpname}-fpm:' \
          +    -e 's:/etc/php-fpm.conf:%{_sysconfdir}/%{phpname}-fpm.conf:' \
          +    -e 's:/usr/sbin:%{_sbindir}:' \
          +    -i $RPM_BUILD_ROOT%{_root_initddir}/%{?scl_prefix}%{phpname}-fpm

    *   Same concept for logrotate configs:

           # LogRotate
          -install -m 755 -d $RPM_BUILD_ROOT%{_origsysconfdir}/logrotate.d
          -install -m 644 %{SOURCE7} $RPM_BUILD_ROOT%{_origsysconfdir}/logrotate.d/php-fpm
          +install -m 755 -d $RPM_BUILD_ROOT%{_root_sysconfdir}/logrotate.d
          +install -m 644 %{SOURCE7} $RPM_BUILD_ROOT%{_root_sysconfdir}/logrotate.d/%{?scl_prefix}%{phpname}-fpm
          +sed -e 's:/var:%{_localstatedir}:' \
          +    -i $RPM_BUILD_ROOT%{_root_sysconfdir}/logrotate.d/%{?scl_prefix}%{phpname}-fpm

    *   While you can `source` the *enable* script to get the SCL binaries in your $PATH, we can also create symlinks with the SCL prefix in system folders. This is ideal if you would only need occasional access to the SCL binary, then you could use `php53-php` instead of `/opt/rh/php53/root/usr/bin/php`:

          +# make the cli commands available in standard root for SCL build
          +%if 0%{?scl:1}
          +install -m 755 -d $RPM_BUILD_ROOT%{_root_bindir}
          +ln -s %{_bindir}/php       $RPM_BUILD_ROOT%{_root_bindir}/%{?scl_prefix}php
          +ln -s %{_bindir}/phar.phar $RPM_BUILD_ROOT%{_root_bindir}/%{?scl_prefix}phar
          +%endif

    *   You have previously seen `if 0%{?scl:1}` as a way to define an if block if an SCL is being built. For single lines, you can use  `%{?scl: --item--}`, like conditionally adding directories:

          -%{_libdir}/httpd/modules/libphp5.so
          -%{_libdir}/httpd/modules/libphp5-zts.so
          +%{?scl: %dir %{_httpd_moddir}}
          +%{_httpd_moddir}/libphp5.so
          +%{?scl: %{_root_httpd_moddir}/libphp5.so}
          +%{_httpd_moddir}/libphp5-zts.so
          +%{?scl: %{_root_httpd_moddir}/libphp5-zts.so}

    *   **TL;DR:** Read through the diff file, but you're making the SPEC file to be compatible with and without SCL build options. This is great because you don't have to maintain a second SPEC file for building SCL packages. There are several conditions that if you are using an SCL, you can have files/scripts outside of the SCL root, like init.d scripts, logrotate configs, symlinks to binaries, etc.
    
*   Apply the diff file as a patch to the AtomiCorp SPEC file. I'm ignoring whitespace because the text editor I used (Sublime Text) removed some spaces for me.

        patch -p0 --ignore-whitespace < php_scl.diff

*   Use `rpmbuild` to build the SRPM. The **--define "scl php53"** option enables the Software Collection (without it, you'd be building PHP packages that install into system default paths), **-bs** will only build a source package, and **--nodeps** will not check for *Requires* or *BuildRequires* on the build system (we'll use `mock` to build the binary RPMs and it will install required packages into the chroot):

    *   CentOS 5: Use MD5 for checksums:

            rpmbuild --define "_source_filedigest_algorithm md5" --define "_binary_filedigest_algorithm md5" --define "scl php53" -bs --nodeps php.spec

    *   CentOS 6:

            rpmbuild --define "scl php53" -bs --nodeps php.spec

*   Use `mock` with the new chroot configurations we created in */etc/mock* earlier:

        mock -v -r epel-6-x86_64-scl-php53 php53-php-5.3.26-19.el6.src.rpm

*   After several minutes, you should have ready-to-install binary packages in */var/lib/mock/epel-6-x86_64-scl-php53/result/*. We'll need some of them to build other packages, like PEAR, APC, Memcache, etc, so we'll add them to our local Yum repo we set up earlier (and this location will serve as where we collect all of our completed RPMs):

        cp -va /var/lib/mock/epel-6-x86_64-scl-php53/result/*.{noarch,x86_64}.rpm ~/rpmbuild/RPMS/repo/el6/x86_64
        cp -va /var/lib/mock/epel-6-x86_64-scl-php53/result/*.src.rpm ~/rpmbuild/RPMS/repo/el6/SRPMS
        createrepo -v -d ~/rpmbuild/RPMS/repo/el6/x86_64
    At this point, you should have several packages in *~/rpmbuild/RPMS/repo/el6/x86_64*. You'll generally want to copy completed .rpm files out of /var/lib/mock as soon as they're finished being built so they aren't destroyed the next time to run `mock` with the same chroot (option *-r*).

## PEAR

*   Get latest php-pear from Red Hat at [http://ftp.redhat.com/redhat/linux/enterprise/6Server/en/os/SRPMS/](http://ftp.redhat.com/redhat/linux/enterprise/6Server/en/os/SRPMS/) and install:

        rpm -ivh php-pear-1.9.4-4.el6.src.rpm

*   Grab [php-pear_scl.diff](/files/php-pear_scl.diff) to use with `patch` to convert the SPEC file for use with SCL:

        curl --remote-name --location http://alanthing.com/files/php-pear_scl.diff

    *   I won't rehash with the same amount of detail as we covered for *php.spec*, but as a quick summary, this patch file will:

    *   Define the SCL package name:

            +%{?scl:%scl_package php-pear}
            +%{!?scl:%global pkg_name %{name}}

    *   Add `%{scl_prefix}` to *Name*, *Requires*, *BuildRequires*, *Obsoletes*, *Conflicts*, and *Provides*. Here's one example:

            -Name: php-pear
            +Name: %{?scl_prefix}php-pear

    *   Add `%{?scl:Requires: %scl_runtime}` so the SCL Base Package is a requirement:

            +%{?scl:Requires: %scl_runtime}

    *   Some macros need to use the system default paths. Remember, `%{_root_*}`-style macros are provided by the *scl-utils-build* package:

            -export PHP_PEAR_SIG_BIN=%{_bindir}/gpg
            +export PHP_PEAR_SIG_BIN=%{_root_bindir}/gpg

    *   The executable scripts for PEAR need to use the actual rpm macro instead of static paths like "/usr":

            +for exe in pear pecl peardev; do
            +    sed -e 's:/usr:%{_prefix}:' \
            +        -i $RPM_BUILD_ROOT%{_bindir}/$exe
            +done

    *   Make symbolic links in the system default directories for the SCL executables:

            +# make the cli commands available in standard root for SCL build
            +%if 0%{?scl:1}
            +install -m 755 -d $RPM_BUILD_ROOT%{_root_bindir}
            +ln -s %{_bindir}/pear      $RPM_BUILD_ROOT%{_root_bindir}/%{scl_prefix}pear
            +ln -s %{_bindir}/pecl      $RPM_BUILD_ROOT%{_root_bindir}/%{scl_prefix}pecl
            +%endif

*   Apply the diff file as a patch to the Red Hat SPEC file:

        patch -p0 --ignore-whitespace < php-pear_scl.diff

*   Use `rpmbuild` to build the SRPM:

    *   CentOS 5:

            rpmbuild --define "_source_filedigest_algorithm md5" --define "_binary_filedigest_algorithm md5" --define "scl php53" -bs --nodeps php-pear.spec

    *   CentOS 6:

            rpmbuild --define "scl php53" -bs --nodeps php-pear.spec

*   Use `mock` build the packages in our SCL chroot:

        mock -v -r epel-6-x86_64-scl-php53 php53-php-pear-1.9.4-4.el6.src.rpm

*   Copy the files to the local Yum repo folder where we're keeping all of our completed RPMs:

        cp -va /var/lib/mock/epel-6-x86_64-scl-php53/result/*.{noarch,x86_64}.rpm ~/rpmbuild/RPMS/repo/el6/x86_64
        cp -va /var/lib/mock/epel-6-x86_64-scl-php53/result/*.src.rpm ~/rpmbuild/RPMS/repo/el6/SRPMS

*   Since APC and Memcache will need *php53-php-pear* as a build dependency, we'll need to update the Yum repo files:

        createrepo -v -d ~/rpmbuild/RPMS/repo/el6/x86_64

## APC

For the remaining packages, I will skip explaining the differences in the SPEC files and simply list the commands. If you look at the diff patch for *php-pecl-apc.spec*, you'll notice that it's very similar to the changes made for PEAR.

*   Get latest php-pecl-apc from Red Hat at [http://ftp.redhat.com/redhat/linux/enterprise/6Server/en/os/SRPMS/](http://ftp.redhat.com/redhat/linux/enterprise/6Server/en/os/SRPMS/) and install:

        rpm -ivh php-pecl-apc-3.1.9-2.el6.src.rpm

*   Grab [php-pecl-apc_scl.diff](/files/php-pecl-apc_scl.diff) to use with `patch` to convert the SPEC file for use with SCL:

        curl --remote-name --location http://alanthing.com/files/php-pecl-apc_scl.diff

    *   Actually, I will point out an important change in this spec file: adding the directory path in front of `php` and `php-config` to ensure we're using the SCL binary and not the system default `php`:

            -%global php_zendabiver %((echo 0; php -i 2>/dev/null | sed -n 's/^PHP Extension => //p') | tail -1)
            -%global php_version %((echo 0; php-config --version 2>/dev/null) | tail -1)
            +%global php_zendabiver %((echo 0; %{_bindir}/php -i 2>/dev/null | sed -n 's/^PHP Extension => //p') | tail -1)
            +%global php_version %((echo 0; %{_bindir}/php-config --version 2>/dev/null) | tail -1)

*   Apply the diff file as a patch to the Red Hat SPEC file:

        patch -p0 --ignore-whitespace < php-pecl-apc_scl.diff

*   Use `rpmbuild` to build the SRPM:

    *   CentOS 5:

            rpmbuild --define "_source_filedigest_algorithm md5" --define "_binary_filedigest_algorithm md5" --define "scl php53" -bs --nodeps php-pecl-apc.spec

    *   CentOS 6:

            rpmbuild --define "scl php53" -bs --nodeps php-pecl-apc.spec

*   Use `mock` build the packages in our SCL chroot:

        mock -v -r epel-6-x86_64-scl-php53 php53-php-pecl-apc-3.1.9-2.el6.src.rpm

*   Copy the files to the local Yum repo folder where we're keeping all of our completed RPMs:

        cp -va /var/lib/mock/epel-6-x86_64-scl-php53/result/*.{noarch,x86_64}.rpm ~/rpmbuild/RPMS/repo/el6/x86_64
        cp -va /var/lib/mock/epel-6-x86_64-scl-php53/result/*.src.rpm ~/rpmbuild/RPMS/repo/el6/SRPMS

*   No other packages require the completed RPMs to build, so we skip the `createrepo` command here and move onto the next page.

## Memcache

We're going to upgrade php-pecl-memcache from 3.0.5 to 3.0.8. Both are considered "beta" on [pecl.php.net](http://pecl.php.net/package/memcache) so it's not like we're losing stability, but we are gaining features.

*   Get latest php-pecl-memcache from Red Hat at [http://ftp.redhat.com/redhat/linux/enterprise/6Server/en/os/SRPMS/](http://ftp.redhat.com/redhat/linux/enterprise/6Server/en/os/SRPMS/) and install:

        rpm -ivh php-pecl-memcache-3.0.5-4.el6.src.rpm

*   Grab [php-pecl-memcache_scl.diff](/files/php-pecl-memcache_scl.diff) to use with `patch` to convert the SPEC file for use with SCL:

        curl --remote-name --location http://alanthing.com/files/php-pecl-memcache_scl.diff

*   Download the [memcache 3.0.8 source](http://pecl.php.net/get/memcache-3.0.8.tgz) from the PECL website into the ~/rpmbuild/SOURCES directory

        curl --output ~/rpmbuild/SOURCES/memcache-3.0.8.tgz --location http://pecl.php.net/get/memcache-3.0.8.tgz

*   Apply the diff file as a patch to the Red Hat SPEC file, which will upgrade 3.0.5 to 3.0.8 and remove the need for 3 patches:

        patch -p0 --ignore-whitespace < php-pecl-memcache_scl.diff

*   Use `rpmbuild` to build the SRPM:

    *   CentOS 5:

            rpmbuild --define "_source_filedigest_algorithm md5" --define "_binary_filedigest_algorithm md5" --define "scl php53" -bs --nodeps php-pecl-memcache.spec

    *   CentOS 6:

            rpmbuild --define "scl php53" -bs --nodeps php-pecl-memcache.spec

*   Use `mock` build the packages in our SCL chroot:

        mock -v -r epel-6-x86_64-scl-php53 php53-php-pecl-memcache-3.0.8-1.el6.src.rpm

*   Copy the files to the local Yum repo folder where we're keeping all of our completed RPMs:

        cp -va /var/lib/mock/epel-6-x86_64-scl-php53/result/*.{noarch,x86_64}.rpm ~/rpmbuild/RPMS/repo/el6/x86_64
        cp -va /var/lib/mock/epel-6-x86_64-scl-php53/result/*.src.rpm ~/rpmbuild/RPMS/repo/el6/SRPMS

## PHP 5.3 SCL Completed

You should now have several binary RPMs in *~/rpmbuild/RPMS/repo/el6/x86_64* ready to be installed as a Software Collection. If you want to distribute your SRPMs, you'll probably want to use the ones generated by `mock` that are in *~/rpmbuild/RPMS/repo/el6/SRPMS*.

## PHP 5.2

PHP 5.2 is quite old, but if 5.3 or higher broke compatibility with old websites, you can install PHP 5.2 as a Software Collection and use PHP-FPM to use only the older version for a specific website. As previously mentioned, I'm using a SRPM that has backported patches (presumably from [https://code.google.com/p/php52-backports/](https://code.google.com/p/php52-backports/)), so it's certainly more up-to-date than [the last 5.2 release from php.net](http://php.net/releases/index.php) (Released: 06 January 2011). 

I won't go into detail of the diff patches since they are largely similar to the changes made on the PHP 5.3 SPECs.

### Software Collection base

*   As with php53, start with [http://people.redhat.com/rcollet/php54/rhel/5/SRPMS/php54-1-2.el5.src.rpm](http://people.redhat.com/rcollet/php54/rhel/5/SRPMS/php54-1-2.el5.src.rpm)

*   Rename or copy php54.spec to php52.spec

*   Set the global variable `scl` to be **php52**:

        %global scl php52

*   Use `rpmbuild` to build the SRPM:

    *   CentOS 5: 
    
            rpmbuild --define "_source_filedigest_algorithm md5" --define "_binary_filedigest_algorithm md5" -bs --nodeps php52.spec

    *   CentOS 6: 

            rpmbuild -bs --nodeps php52.spec

*   Build with mock (repeat el6 instructions for el5):

        mock -v -r epel-6-x86_64 php52-1-2.el5.src.rpm

*   Copy resulting files to local folder and create a local Yum repo to be used later (repeat el6 instructions for el5):

        mkdir -vp ~/rpmbuild/RPMS/repo/el6/{x86_64,SRPMS}
        cp -va /var/lib/mock/epel-6-x86_64/result/*.{noarch,x86_64}.rpm ~/rpmbuild/RPMS/repo/el6/x86_64
        cp -va /var/lib/mock/epel-6-x86_64/result/*.src.rpm ~/rpmbuild/RPMS/repo/el6/SRPMS
        createrepo -v -d ~/rpmbuild/RPMS/repo/el6/x86_64

### Mock

These steps are nearly identical are for php53, but using php52 instead. As mentioned previously, if you need to repeat something for CentOS 5, use the CentOS 6 and change 6 to 5, el5 to el6, etc.

*   Change to the mock configuration directory:

        cd /etc/mock

*   Copy the default EPEL files to new ones specific to our SCL:

        cp -a epel-6-x86_64.cfg epel-6-x86_64-scl-php52.cfg

*   Edit *epel-6-x86_64-scl-php52.cfg* and:

    *   Add **-scl-php53** to the existing value of *config_opts['root']*:

            config_opts['root'] = 'epel-6-x86_64-scl-php52'

    *   CentOS 5 and 6 have different options for *config_opts['chroot_setup_cmd']*:

        *   CentOS 5: Add **scl-utils-build php52-build** the following to the existing value:

                config_opts['chroot_setup_cmd'] = 'install buildsys-build scl-utils-build php52-build'

        *   CentOS 6: Change from 'groupinstall buildsys-build' to **install @buildsys-build scl-utils-build php52-build**:

                config_opts['chroot_setup_cmd'] = 'install @buildsys-build scl-utils-build php52-build'

*   Add the local repo where you left the files from the section above as a new repo; I add mine above *[base]*. **Change /home/alan to the appropriate path for your system!**

        [localrepo-6-x86_64]
        name=localrepo-6-x86_64
        enabled=1
        baseurl=file:///home/alan/rpmbuild/RPMS/repo/el6/x86_64

### PHP

*   Use latest php-5.2.x SRPM from [http://centos.alt.ru/pub/repository/centos/5/SRPMS/](http://centos.alt.ru/pub/repository/centos/5/SRPMS/)

*   Install the SRPM

        rpm -ivh php-5.2.17-29.el5.src.rpm

*   Several changes are needed to add the SCL macros to the SPEC file; see the [Red Hat documentation](https://access.redhat.com/site/documentation/en-US/Red_Hat_Developer_Toolset/1/html/Software_Collections_Guide/sect-Converting_a_Conventional_Spec_File.html). Download my patch file for a full list of changes needed to convert the alt.ru PHP SPEC to one that can be used with and without SCL options: [php52_scl.diff](/files/php52_scl.diff)

        curl --remote-name --location http://alanthing.com/files/php52_scl.diff

*   Apply the diff file as a patch to the alt.ru SPEC file:

        patch -p0 --ignore-whitespace < php52_scl.diff

*   Use `rpmbuild` to build the SRPM:

    *   CentOS 5:

            rpmbuild --define "_source_filedigest_algorithm md5" --define "_binary_filedigest_algorithm md5" --define "scl php52" -bs --nodeps php.spec

    *   CentOS 6:

            rpmbuild --define "scl php52" -bs --nodeps php.spec

*   Use `mock` with the new chroot configurations we created in */etc/mock* earlier:

        mock -v -r epel-6-x86_64-scl-php52 php52-php-5.2.17-29.el6.src.rpm

*   After `mock` is completed, copy the files to the local repo folder. Like before, we'll need some of these packages to build other packages:

        mkdir -vp ~/rpmbuild/RPMS/repo/el6/{SRPMS,x86_64}
        cp -va /var/lib/mock/epel-6-x86_64-scl-php52/result/*.{noarch,x86_64}.rpm ~/rpmbuild/RPMS/repo/el6/x86_64
        cp -va /var/lib/mock/epel-6-x86_64-scl-php52/result/*.src.rpm ~/rpmbuild/RPMS/repo/el6/SRPMS
        createrepo -v -d ~/rpmbuild/RPMS/repo/el6/x86_64

### PEAR, APC, and Memcache

You can repeat the same steps for php-pear, php-pecl-apc, and php-pecl-memcache as above for PHP 5.3. The only difference would be to change the SCL name during the `rpmbuild` and `mock` steps. Here are all of the steps without explanation:

#### PEAR

```bash
cd ~/rpmbuild/SRPMS
curl --remote-name --location http://ftp.redhat.com/redhat/linux/enterprise/6Server/en/os/SRPMS/php-pear-1.9.4-4.el6.src.rpm
rpm -ivh php-pear-1.9.4-4.el6.src.rpm
cd ~/rpmbuild/SPECS
curl --remote-name --location http://alanthing.com/files/php-pear_scl.diff
patch -p0 --ignore-whitespace < php-pear_scl.diff
rpmbuild --define "scl php52" -bs --nodeps php-pear.spec
mock -v -r epel-6-x86_64-scl-php52 php52-php-pear-1.9.4-4.el6.src.rpm
cp -va /var/lib/mock/epel-6-x86_64-scl-php52/result/*.{noarch,x86_64}.rpm ~/rpmbuild/RPMS/repo/el6/x86_64
cp -va /var/lib/mock/epel-6-x86_64-scl-php52/result/*.src.rpm ~/rpmbuild/RPMS/repo/el6/SRPMS
createrepo -v -d ~/rpmbuild/RPMS/repo/el6/x86_64
```

#### APC

```bash
cd ~/rpmbuild/SRPMS
curl --remote-name --location http://ftp.redhat.com/redhat/linux/enterprise/6Server/en/os/SRPMS/php-pecl-apc-3.1.9-2.el6.src.rpm
rpm -ivh php-pecl-apc-3.1.9-2.el6.src.rpm
cd ~/rpmbuild/SPECS
curl --remote-name --location http://alanthing.com/files/php-pecl-apc_scl.diff
patch -p0 --ignore-whitespace < php-pecl-apc_scl.diff
rpmbuild --define "scl php52" -bs --nodeps php-pecl-apc.spec
mock -v -r epel-6-x86_64-scl-php52 php52-php-pecl-apc-3.1.9-2.el6.src.rpm
cp -va /var/lib/mock/epel-6-x86_64-scl-php52/result/*.{noarch,x86_64}.rpm ~/rpmbuild/RPMS/repo/el6/x86_64
cp -va /var/lib/mock/epel-6-x86_64-scl-php52/result/*.src.rpm ~/rpmbuild/RPMS/repo/el6/SRPMS
```

#### Memcache

```bash
cd ~/rpmbuild/SRPMS
curl --remote-name --location http://ftp.redhat.com/redhat/linux/enterprise/6Server/en/os/SRPMS/php-pecl-memcache-3.0.5-4.el6.src.rpm
rpm -ivh php-pecl-memcache-3.0.5-4.el6.src.rpm
cd ~/rpmbuild/SPECS
curl --remote-name --location http://alanthing.com/files/php-pecl-memcache_scl.diff
curl --output ~/rpmbuild/SOURCES/memcache-3.0.8.tgz --location http://pecl.php.net/get/memcache-3.0.8.tgz
patch -p0 --ignore-whitespace < php-pecl-memcache_scl.diff
rpmbuild --define "scl php52" -bs --nodeps php-pecl-memcache.spec
mock -v -r epel-6-x86_64-scl-php52 php52-php-pecl-memcache-3.0.8-1.el6.src.rpm
cp -va /var/lib/mock/epel-6-x86_64-scl-php52/result/*.{noarch,x86_64}.rpm ~/rpmbuild/RPMS/repo/el6/x86_64
cp -va /var/lib/mock/epel-6-x86_64-scl-php52/result/*.src.rpm ~/rpmbuild/RPMS/repo/el6/SRPMS
```

## Conclusion

It does seem like a little bit of work to convert a SPEC file for SCL compatibility, but once the work is done, incremental updates are easy, and you can use the same SPEC to build a non-SCL package as well. I wouldn't be surprised if Red Hat begins to create SCL-compatible SPEC files for major applications in RHEL 7 and beyond because of the flexibility it offers, just see their [Red Hat Software Collections 1.0 Beta](http://www.redhat.com/about/news/archive/2013/6/red-hat-software-collections-1.0-beta-now-available). Unfortunately, it's difficult to fully grasp how to build one, so hopefully this guide has been helpful. 

Personally, after having some trouble quickly switching from MySQL to MariaDB or Percona, I'll probably look to rebuild those as Software Collections instead of replacing the default packages, but that's another blog post!