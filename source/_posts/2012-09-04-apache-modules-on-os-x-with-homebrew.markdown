---
layout: post
title: "Apache modules on OS X with Homebrew"
date: 2012-09-04 11:27
comments: true
categories: 
---

I have long been a big fan of [Homebrew](http://mxcl.github.com/homebrew), a way of adding open-source software to OS X without much of the overhead and systems link [MacPorts](http://www.macports.org) and [Fink](http://www.finkproject.org) introduce. It's quickly become my favorite way to do PHP-based development on a Mac, and I've [blogged](http://echodittolabs.org/blog/2012/04/os-x-107-lion-development-nginx-php-mariadb-homebrew) [before](http://echodittolabs.org/blog/2011/09/os-x-107-lion-development-native-apache-php-homebrew-mysql-or-mariadb) about various ways to integrate Homebrew-based software into your workflow.

In preparing new documentation for using Homebrew on 10.7 and 10.8 (coming soon to [my EchoDitto blog](http://echodittolabs.org/blogs/alan-ivey)), I wanted to find a way to run either [mod_suexec](http://httpd.apache.org/docs/2.2/mod/mod_suexec.html) or [mod_fastcgi](http://www.fastcgi.com/mod_fastcgi/docs/mod_fastcgi.html) so I could use the built-in version of Apache without having to change folder ownership in random places where the web server needed to modify files. Homebrew did not have any formulas, but I found that [Lifepillar](http://github.com/lifepillar) had written formulas [(mod_fastcgi)](https://github.com/mxcl/homebrew/pull/12093/files) [(mod_suexec)](https://github.com/mxcl/homebrew/pull/12091) for them both and was hoping to get them into Homebrew.

Driven solely by my desire to have my documentation easier to follow, I set up a [brew tap](https://github.com/mxcl/homebrew/wiki/Homebrew-0.9) for these two Apache modules. Shortly after adding it to the list of [interesting taps and branches](https://github.com/mxcl/homebrew/wiki/Interesting-Taps-&-Branches), the decision was made to [incorporate the repository under the Homebrew organization](https://github.com/mxcl/homebrew/issues/14622) and have a place just for Apache modules.

As of today, here are the modules available in the tap:

* [mod_fastcgi](https://github.com/Homebrew/homebrew-apache/blob/master/mod_fastcgi.rb) - written by [lifepillar](https://github.com/lifepillar)
* [mod_suexec](https://github.com/Homebrew/homebrew-apache/blob/master/mod_suexec.rb) - written by [lifepillar](https://github.com/lifepillar), 10.8 work by [me](https://github.com/alanthing)
* [mod_fcgid](https://github.com/Homebrew/homebrew-apache/blob/master/mod_fcgid.rb) - written by [rjocoleman](https://github.com/rjocoleman)
* [mod_bonjour](https://github.com/Homebrew/homebrew-apache/blob/master/mod_bonjour.rb) - written by [joemaller](https://github.com/joemaller)
* [mod_python](https://github.com/Homebrew/homebrew-apache/blob/master/mod_python.rb) and [mod_wsgi](https://github.com/Homebrew/homebrew-apache/blob/master/mod_wsgi.rb) - copied from main Homebrew formulas

If you have any problems or questions, let me know in the issue queue! Or write a formula and submit a pull request!
