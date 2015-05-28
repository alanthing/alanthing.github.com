---
layout: post
title: "OS X 10.7 Lion Development: MacPorts"
date: 2011-10-11 14:13
comments: true
categories:
---

*This post originally featured on the [Echo &amp; Co. blog](http://echo.co/blog/os-x-107-lion-development-macports)*.

OS X Lion comes with most of the tools you would need to do "MAMP" (Mac OS X, Apache, MySQL/MariaDB, PHP) development, as outlined in my [previous](http://echodittolabs.org/blog/2011/08/os-x-107-lion-development-native-mamp-mysql-installer) [posts](http://echodittolabs.org/blog/2011/09/os-x-107-lion-development-native-apache-php-homebrew-mysql-or-mariadb) once you add a database. So then why would you want to use MacPorts? Setting your development environment up in MacPorts isolates the binaries, libraries, and configuration files, completely separate from the existing OS X install (with the exception of startup scripts). You can also tweak the configuration files on your own, apply your own patches, and apply updates that MacPorts may get before Apple pushes them. It will take more time because you'll be compiling everything, but you have all of the control. Read on for how to get things set up. 

## Prerequisites

We'll going to be compiling a lot of source code so you'll need to have [Xcode 4](http://itunes.apple.com/us/app/xcode/id448457090) installed. I initially tried with the [OS X GCC Installer](https://github.com/kennethreitz/osx-gcc-installer) but eventually ran into a problem where a portfile was expecting an Xcode binary to check to a prerequisite. I'm sure some hacking could've resolved it to avoid having to install the very large Xcode package, but at the end of the day it's better to just know that it'll work as expected. 

Once you have Xcode 4 installed, you'll need to [install MacPorts](http://www.macports.org/install.php). It's a simple click-through installer. Come back to this guide once MacPorts is ready. 

Note that for all commands before that are starting with a $, the dollar sign is showing a command-line prompt in [Terminal](http://www.apple.com/macosx/apps/all.html#terminal), and you should not actually type it as part of the commands. 

## Install packages

Update the ports tree to get the latest from the MacPorts server in case the installer had an older tree:

```bash
sudo /opt/local/bin/port selfupdate
```

All of the packages that we need can be installed with a single command:

```bash
sudo /opt/local/bin/port install apache2 php5 php5-mysql mysql5-server php5-gd php5-mbstring php5-apc lynx phpmyadmin
```

Compiling all of these tools will take a while, so grab a book and come back when the command is completed. 

## Configure PHP

The following will set up PHP with some development-friendly settings. 

```bash
$ sudo cp -av /opt/local/etc/php5/php.ini-development /opt/local/etc/php5/php.ini
$ sudo sed -i "" 's/\(default_socket[ ]\{0,1\}=\)/\1 \/opt\/local\/var\/run\/mysql5\/mysqld.sock/g' /opt/local/etc/php5/php.ini
$ sudo sh -c "cat >> /opt/local/etc/php5/php.ini <<'EOF'


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
; Original - ;date.timezone =
date.timezone = 'America/New_York'
EOF"
```

Now configure the APC extension (installed above with php5-apc) which will increase PHP performance dramatically.

```bash
$ sudo sh -c "cat > /opt/local/var/db/php5/apc-config.ini <<'EOF'
[apc]
apc.enabled = 1
apc.shm_segments = 1
apc.shm_size = 96M
apc.cache_by_default = 1
apc.stat = 1
apc.rfc1867 = 1
apc.stat = 7200
EOF"
```

## Configure Apache

Start by setting up PHP to work with Apache properly:

```bash
$ sudo cp -av /opt/local/apache2/conf/httpd.conf /opt/local/apache2/conf/httpd.conf-default
$ sudo /opt/local/apache2/bin/apxs -a -e -n "php5" /opt/local/apache2/modules/libphp5.so
$ sudo bash -c "cat >> /opt/local/apache2/conf/httpd.conf <<'EOF'

## Add mod_php information
Include conf/extra/mod_php.conf
# Add index.php to the list of files that will be served as directory indexes.
<IfModule dir_module>
  DirectoryIndex index.php index.html
</IfModule>
EOF"
```

Now we'll move httpd-vhosts.conf to ~/Sites for easy editing of new virtual hosts, as well as create a ~/Sites/logs folder:

```bash
$ sudo mv -v /opt/local/apache2/conf/extra/httpd-vhosts.conf /opt/local/apache2/conf/extra/httpd-vhosts.conf-default </br>
$ [ ! -d ~/Sites ] && mkdir -v ~/Sites
$ cp -av /opt/local/apache2/conf/extra/httpd-vhosts.conf-default ~/Sites/httpd-vhosts.conf
$ sudo ln -s ~/Sites/httpd-vhosts.conf /opt/local/apache2/conf/extra/httpd-vhosts.conf
$ sudo sed -i "" 's/\#\(.*httpd-vhosts\.conf\)/\1/' /opt/local/apache2/conf/httpd.conf
$ [ ! -d ~/Sites/logs ] && mkdir ~/Sites/logs
$ sudo perl -pi -e 'BEGIN{undef $/;} s/\<VirtualHost .*\n//sg;' ${USERHOME}/Sites/httpd-vhosts.conf </br>
$ sudo sh -c "cat >> ~/Sites/httpd-vhosts.conf <<'EOF' </br>
 </br>
## Change /Users/name to the path of your home folder </br>
#<VirtualHost *:80> </br>
#  ServerName project.local </br>
#  CustomLog "/Users/name/Sites/logs/project.local-access_log" combined </br>
#  ErrorLog "/Users/name/Sites/logs/project.local-error_log" </br>
#  DocumentRoot "/Users/name/Sites/project.local" </br>
#</VirtualHost> </br>
EOF"
```

Allow userdirs so the MacPorts Apache "feels" more like the built-in Apache:

```bash
sudo sed -i "" 's/\#\(.*httpd-userdir\.conf\)/\1/' /opt/local/apache2/conf/httpd.conf
```

Start Apache and set it to load on boot:

```bash
sudo /opt/local/bin/port load apache2
```

## Configure MySQL

Set up the configuration file:

```bash
sudo cp -av /opt/local/share/mysql5/mysql/my-small.cnf /opt/local/etc/mysql5/my.cnf
sudo sed -i "" 's/max_allowed_packet = 1.*M/max_allowed_packet = 1G/g' /opt/local/etc/mysql5/my.cnf
```

Initialize MySQL and run the secure installation script:
```bash
sudo -u _mysql /opt/local/bin/mysql_install_db5
sudo /opt/local/bin/port load mysql5-server
sudo /opt/local/lib/mysql5/bin/mysql_secure_installation
```

## Configure phpMyAdmin

Add a config file for Apache and reload Apache:

```bash
$ sudo sh -c "cat >> /opt/local/apache2/conf/httpd.conf <<'EOF'

# phpMyAdmin
Include conf/extra/phpmyadmin.conf
EOF"
$ sudo sh -c "cat >> /opt/local/apache2/conf/extra/phpmyadmin.conf <<'EOF'
AliasMatch ^/phpmyadmin(?:/)?(/.*)?$ /opt/local/www/phpmyadmin$1
AliasMatch ^/phpMyAdmin(?:/)?(/.*)?$ /opt/local/www/phpmyadmin$1

<Directory "/opt/local/www/phpmyadmin">
 Options -Indexes
 AllowOverride None
 Order allow,deny
 Allow from all

 LanguagePriority en de es fr ja ko pt-br ru
 ForceLanguagePriority Prefer Fallback
</Directory>
EOF"
$ sudo /opt/local/bin/port unload apache2
$ sudo /opt/local/bin/port load apache2
```

Basic set up of phpMyAdmin:

```bash
sudo sed -i "" "s/blowfish_secret\'\] = \'/blowfish_secret\'\] = \'`cat /dev/urandom | strings | grep -o '[[:alnum:]]' | head -n 50 | tr -d '\n'`/" /opt/local/www/phpmyadmin/config.inc.php
sudo /opt/local/bin/mysql5 -uroot -proot < /opt/local/www/phpmyadmin/scripts/create_tables.sql
sudo /opt/local/bin/mysql5 -uroot -proot -e"CREATE USER 'pma'@'localhost' IDENTIFIED BY 'pmapass'; GRANT SELECT, INSERT, DELETE, UPDATE ON phpmyadmin.* TO pma@localhost;"
```

## Hooray!

Now you can edit ~/Sites/httpd-vhosts.conf and add a new virtual host. You'd need to reload Apache after doing so by running `sudo port unload apache2 && sudo port load apache2`. phpMyAdmin or the mysql5 binary should provide you a way to create new databases and database users and you can then set up a local site.

If you find MacPorts too heavy a separation from OSX, or too slow to compile, you should check out my previous blog posts on setting up a MAMP environment with as many built-in tools as possible only by adding a [MySQL installer](http://echodittolabs.org/blog/2011/08/os-x-107-lion-development-native-mamp-mysql-installer) or [MySQL/MariaDB via Homebrew](http://echodittolabs.org/blog/2011/09/os-x-107-lion-development-native-apache-php-homebrew-mysql-or-mariadb).