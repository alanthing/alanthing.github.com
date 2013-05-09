---
layout: post
title: "Speed up PHP on NFS with turbo_realpath"
date: 2013-05-09 11:50
comments: true
categories: 
---

If you run a website based on PHP, and have your source files on a network file system like NFS, OCFS2, or GlusterFS, and combine it with PHP's [open_basedir](http://www.php.net/manual/en/ini.core.php#ini.open-basedir) protection, you'll quickly notice that the performance will degrade substantially. Normally, PHP can cache various path locations it learns after processing [include_once](http://us2.php.net/manual/en/function.include-once.php) and [require_once](http://us2.php.net/manual/en/function.require-once.php) calls via the [realpath_cache](http://www.php.net/manual/en/ini.core.php#ini.realpath-cache-size). There's a [bug in PHP](https://bugs.php.net/bug.php?id=52312) that effectively disables the realpath_cache entirely when combined with open_basedir. Popular PHP applications with Drupal and WordPress make heavy use of these functions to include other files, so you would very quickly notice the drop in performance in this scenario. If you want to isolate your websites from each other (or from the rest of the operating system), how can you retain any shred of performance?

This is where [Artur Graniszewski](http://php.webtutor.pl/)'s [turbo_realpath](http://php.webtutor.pl/en/2011/07/12/running-php-on-nfs-version-1-2-of-turbo_realpath-extension/) extension really comes in handy. I won't retype his installation instructions, so follow the previous link to get it installed manually. 

If you're running CentOS 5 or CentOS 6, check out [yum.echoditto.com](http://yum.echoditto.com/) and you'll find source and compiled RPMs that will install alongside the RedHat/CentOS-supplied PHP packages. The RPM will create a basic configuration file at `/etc/php.d/turbo_realpath.ini`. Essentially, it enables the PHP module but defaults all settings off, so you will need to read the comments (taken from [Artur's most recent post on turbo_realpath](http://php.webtutor.pl/en/2011/07/12/running-php-on-nfs-version-1-2-of-turbo_realpath-extension/)) to determine how you want to use it.

# Configuration

We frequently use turbo_realpath on a per-VirtualHost basis with Apache 2.2 and mod_php. If you use PHP-FPM, you can apply similar settings in your FPM pool configuration files. If you install our RPM and don't edit `/etc/php.d/turbo_realpath.ini`, add something similar to the following to each VirtualHost:

```apache
<IfModule php5_module>
  php_admin_value realpath_cache_basedir "/var/www/vhosts/domain.com:/usr/share/pear:/usr/share/php:/usr/lib64/php:/usr/lib/php:/tmp:/var/tmp"
</IfModule>
```

This is effectively the same using `open_basedir`; any directories referenced in `realpath_cache_basedir` will be the only ones the website is allowed to access, and they will be cached as determined by the [realpath_cache_size](http://www.php.net/manual/en/ini.core.php#ini.realpath-cache-size) and [realpath_cache_ttl](http://www.php.net/manual/en/ini.core.php#ini.realpath-cache-ttl). If you look in `php.ini`, you may notice the default values for these are:

```ini
; Determines the size of the realpath cache to be used by PHP. This value should
; http://www.php.net/manual/en/ini.core.php#ini.realpath-cache-size
realpath_cache_size = 16k

; Duration of time, in seconds for which to cache realpath information for a given
; http://www.php.net/manual/en/ini.core.php#ini.realpath-cache-ttl
realpath_cache_ttl = 120
```

You may want to increase these if you're finding your website is still not loading quickly. On our systems, we have bumped the `realpath_cache_size` and `realpath_cache_ttl` settings up to `1m` and `300`, respectively.

# Speed and Security! 

With turbo_realpath enabled, `realpath_cache_basedir` set to appropriate `open_basedir`-like values, and `realpath_cache_size` and `realpath_cache_ttl` increased from defaults, we're able to have isolated PHP sites and have better performance by caching the locations of included/required files effectively. Hopefully, our RPMs will help you on your system for a quick installation of the excellent turbo_realpath module!

# References

* [Running PHP on NFS: huge performance problems and one simple solution.](http://php.webtutor.pl/en/2011/06/02/running-php-on-nfs-huge-performance-problems-and-one-simple-solution/)
* [Running PHP on NFS: new version of turbo_realpath extension](http://php.webtutor.pl/en/2011/07/01/running-php-on-nfs-new-version-of-turbo_realpath-extension/)
* [Running PHP on NFS: version 1.2 of turbo_realpath extension](http://php.webtutor.pl/en/2011/07/12/running-php-on-nfs-version-1-2-of-turbo_realpath-extension/)