---
layout: post
title: "OS X 10.7 Lion Development: Native Apache & PHP with Homebrew MySQL or MariaDB"
date: 2011-09-07 17:45
comments: true
categories:
---

*This post originally featured on the [Echo &amp; Co. blog](http://echo.co/blog/os-x-107-lion-development-native-apache-php-homebrew-mysql-or-mariadb)*.

OS X Lion ships with Apache and PHP, which both require a little bit of tweaking to get fully-functional for "MAMP" local development. The one thing Lion does not ship with is a database. This will be very similar to my [previous post on local development](http://echodittolabs.org/blog/2011/08/os-x-107-lion-development-native-mamp-mysql-installer) but this time we'll be using Homebrew to install either MySQL or MariaDB for the database. Since we'll be using a compiler for Homebrew, I'll also cover how to add APC and other PECL modules that you can add to OS X. 

Note that for all commands before that are starting with a $, the dollar sign is showing a command-line prompt in [Terminal](http://www.apple.com/macosx/apps/all.html#terminal), and you should not actually type it as part of the commands.

## Apache

We'll set things up so we won't need sudo often in the future, and so we can manage multiple VirtualHosts. We'll keep the Apache information and our website roots in ~/Sites, and Apache logs in ~/Sites/logs

```bash
[ ! -d ~/Sites ] && mkdir ~/Sites 
touch ~/Sites/httpd-vhosts.conf 
sudo ln -s ~/Sites/httpd-vhosts.conf /etc/apache2/other 
mkdir ~/Sites/logs 
chmod 0777 ~/Sites/logs
```

Edit the new **~/Sites/httpd-vhosts.conf** file and add the following. Note that you'll need to change all instances of "/Users/name" to your actual home folder path.

```apache
# 
# Use name-based virtual hosting. 
# 
NameVirtualHost *:80 

# 
# Set up permissions for VirtualHosts in ~/Sites 
# 
<Directory "/Users/name/Sites"> 
    Options Indexes FollowSymLinks MultiViews 
    AllowOverride All 
    Order allow,deny 
    Allow from all 
</Directory> 

# For http://localhost in the OS X default location 
<VirtualHost _default_:80> 
    ServerName localhost 
    DocumentRoot /Library/WebServer/Documents 
</VirtualHost> 

# 
# VirtualHosts below 
# 

## Template 
#<VirtualHost *:80> 
#    ServerName domain.local 
#    CustomLog "/Users/name/Sites/logs/domain.local-access_log" combined 
#    ErrorLog "/Users/name/Sites/logs/domain.local-error_log" 
#    DocumentRoot "/Users/name/Sites/domain.local" 
#</VirtualHost>
```

Launch **System Preferences** and go to **Sharing** and toggle **Web Sharing** off and on so it's started and reloaded with the new settings. Then click on the blue underlined link under *"Your computer's website is available at this address:"* to ensure Apache is working correctly, and you should see text saying "It works!"

To add a site, duplicate the &lt;VirtualHost&gt; section under the Template, remove the comments, and edit appropriately. Edit /etc/hosts (or use [Gas Mask](http://code.google.com/p/gmask/)) and add an entry for 127.0.0.1 for your project.local, and make the appropriate change in the conf file, along with the correct DocumentRoot and Log file locations. See [Apache's documentation](http://httpd.apache.org/docs/2.2/vhosts/name-based.html) for more information.

## PHP

OS X has shipped with PHP for quite some time, but not had it enabled by default. We'll enable mod_php for Apache, and also set up an /etc/php.ini file based on the default one shipped with Lion with some development-friendly changes. Change your timezone as needed.

```bash
$ sudo sh -c "grep php /etc/apache2/httpd.conf|grep LoadModule|cut -d'#' -f2 > /etc/apache2/other/php5-loadmodule.conf" 
$ sudo cp -a /etc/php.ini.default /etc/php.ini 
$ sudo sh -c "cat >> /etc/php.ini <<'EOF' 


;; 
;; User customizations below 
;; 

; Original - memory_limit = 128M 
memory_limit = 196M 
; Original - post_max_size = 8M 
post_max_size = 200M 
; Original - upload_max_filesize = 2M 
upload_max_filesize = 100M 
; Original - default_socket_timeout = 60 
default_socket_timeout = 600 
; Original - max_execution_time = 30 
max_execution_time = 300 
; Original - max_input_time = 60 
max_input_time = 600 
; Original - display_errors = Off 
display_errors = on 
; Original - display_startup_errors = Off 
display_startup_errors = on 
; Original - ;date.timezone = 
date.timezone = 'America/New_York' 
EOF"
```

**Optional:** Lion ships with PEAR but not installed, whereas Snow Leopard it was installed. This will install PEAR from the phar archive, upgrade it, and add 'pear' and 'pecl' aliases to your shell.

```bash
$ sudo /usr/bin/php /usr/lib/php/install-pear-nozlib.phar 
$ cat >> ~/.bashrc <<'EOF' 

alias pear="php /usr/lib/php/pear/pearcmd.php" 
alias pecl="php /usr/lib/php/pear/peclcmd.php" 
EOF 
$ . ~/.bashrc 
$ sudo pear channel-update pear.php.net 
$ sudo pecl channel-update pecl.php.net 
$ sudo pear upgrade --force pear 
$ sudo pear upgrade 
$ sudo pecl upgrade 
$ sudo sh -c "cat >> /etc/php.ini <<'EOF' 
; Original - ;include_path = ".:/php/includes" 
include_path = ".:/usr/lib/php/pear" 
EOF"
```

We'll cover APC and other PECL modules after we have Homebrew up and running.

Toggle **Web Sharing** in **System Preferences &gt; Sharing** for the new PHP options to take effect.

## Homebrew

Hop onto the Mac App Store and download Xcode 4. It's free! When it's finished downloading, you'll have an Application called **Install Xcode** in the Applications folder, so run that and click through and you'll be all set. If it seems like overkill since we're not covering iOS or Mac app development, Xcode ships with the compiler we need for Homebrew and installing PECL modules.

To install Homebrew, go to https://github.com/mxcl/homebrew/wiki/installation and run the ruby script in Terminal. If you get the error message: *Error: Cannot write to /usr/local*, run:

```bash
sudo chmod g+w /usr/local
sudo chgrp staff /usr/local
```

## MariaDB / MySQL

[MariaDB](http://mariadb.org/) is a fork of MySQL that has [additional features](http://kb.askmonty.org/en/mariadb-versus-mysql) and is a drop-in replacement for MySQL. MariaDB is [supported by Drupal 7](http://drupal.org/node/861192), and works with Drupal 6 as well. Make the choice to go with either MariaDB or MySQL and continue; I'll cover how to setup and use both.

### MySQL

To install MySQL, simply run:

```bash
brew install mysql
```

cmake is a dependency of MySQL, and cmake needs java, so if you get a pop-up asking you to install a Java runtime, be sure to click Install and proceed. Grab a book because compiling MySQL and its dependencies will take several minutes. Now, to configure MySQL (includes raising packet limits for easier use in a non-production scenario) and start:

```bash
unset TMPDIR
mysql_install_db --verbose --user=`whoami` --basedir="$(brew --prefix mysql)" --datadir=/usr/local/var/mysql --tmpdir=/tmp 
cp $(brew --prefix mysql)/support-files/my-small.cnf /usr/local/var/mysql/my.cnf
sed -i "" 's/max_allowed_packet = 1.*M/max_allowed_packet = 2G/g' /usr/local/var/mysql/my.cnf 
[ ! -d ~/Library/LaunchAgents ] && mkdir ~/Library/LaunchAgents
```

### MariaDB

To install MariaDB, run:

```bash
brew install mariadb pidof
```

*pidof* is referenced in some of the MariaDB scripts but is not listed as a prerequisite in MariaDB's Homebrew formula, so we need to install it. To configure and start:

```bash
unset TMPDIR
mysql_install_db --verbose --user=`whoami` --basedir="$(brew --prefix mariadb)" --datadir=/usr/local/var/mysql --tmpdir=/tmp
cp $(brew --prefix mariadb)/share/mysql/my-small.cnf /usr/local/var/mysql/my.cnf
sed -i "" 's/max_allowed_packet = 1.*M/max_allowed_packet = 2G/g' /usr/local/var/mysql/my.cnf 
[ ! -d ~/Library/LaunchAgents ] && mkdir ~/Library/LaunchAgents
```

## APC and other PECL modules

PECL modules require compiling, which is why I excluded it from my previous blog post because I did not require Xcode to be installed. An easy one is **uploadprogress** which is often recommended for Drupal development:

```bash
$ sudo pecl install uploadprogress 
$ sudo sh -c "cat >> /etc/php.ini <<'EOF' 

; Enable PECL extension uploadprogress 
extension=uploadprogress.so 
EOF"
```

APC is easy too, but requires you to install pcre libraries. With Homebrew, this is easy, and we'll add some basic APC parameters:

```bash
$ brew install pcre 
$ sudo pecl install apc 
$ sudo sh -c "cat >> /etc/php.ini <<'EOF' 

; Enable PECL extension APC 
extension=apc.so 
apc.enabled = 1 
apc.shm_segments = 1 
apc.shm_size = 96M 
apc.cache_by_default = 1 
apc.stat = 1 
apc.rfc1867 = 1 
EOF"
```

Non-PECL extensions, like mcrypt, require some additional work. Check out [this guide for building mcrypt](http://blog.rogeriopvl.com/archives/php-mcrypt-in-snow-leopard-with-homebrew/).

## Make PHP and MariaDB / MySQL Play Nice

If you were to run `php -i|egrep 'mysql.*default_socket'` you would see that PHP was compiled to expect the MySQL socket file in /var/mysql, but MariaDB / MySQL will place it in /tmp. This easiest fix is to tell PHP to look in /tmp:

```bash
sudo sed -i "" 's|/var/mysql/mysql\.sock|/tmp/mysql.sock|g' /etc/php.ini
```

Toggle **Web Sharing** in **System Preferences &gt; Sharing** for the new PHP options to take effect.

## Hooray!

You should now be all set to keep adding more VirtualHosts in httpd-vhosts.conf and begin development on your local machine. Stay tuned for instructions on how to use [MacPorts](http://www.macports.org/)!

**Update:** If you're interested in leveraging more from Homebrew, like replacing Apache and PHP all from Homebrew, check out the [homebrew-alt repository](https://github.com/adamv/homebrew-alt) on GitHub. Newer versions of Homebrew allow you to install formulae from URLs or alternate folders, and homebrew-alt provides formulae that replace built-in OS X applications (something official formulae will not do). For example, you could install PHP with PHP-FPM, something the OS X-provided PHP does not offer. Have fun exploring!