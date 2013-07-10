---
layout: post
title: "Native local development environment in OS X"
date: 2010-10-15 14:46
comments: true
categories:
---

**UPDATE:** [A newer version of this has been posted here](/blog/2011/08/24/os-x-107-lion-development-native-mamp-mysql-installer/).

Apple OS X comes with Apache and PHP built-in but need some tweaking to work. It also does not come with MySQL. Because of this, many developers have chosen to use MacPorts, Homebrew, or MAMP to install new binaries for Apache, PHP, and MySQL. However, doing this means your system would have multiple copies of Apache and PHP on your machine, and could create conflicts depending on how your built-in tools are configured. This tutorial will show you how to get the built-in versions of Apache and PHP running with an easy to install version of MySQL.

Additionally, it will show how to set up and install Drush, and phpMyAdmin. Setting up a new website is as easy as editing a single text file and adding a line to your */etc/hosts* file. All of our changes will avoid editing default configuration files, either those provided by Apple in OS X or from downloaded packages, so that future updates will not break our customizations.

## Apache ##

* Apple uses what appears to be the FreeBSD "version" of Apache 2.2, and includes various sample files. One of the sample files is a virtual hosts file that we will copy to our *~/Sites* folder for easy editing. Launch the terminal and get started:

```bash
cp /etc/apache2/extra/httpd-vhosts.conf ~/Sites
```

* In order to prevent future major OS updates from breaking our configuration, we'll be editing as few existing files as possible. Luckily, by default, OS X's *httpd.conf* file (the main configuration file for Apache) includes all *.conf files in another folder, */etc/apache2/other*:

```bash
$ tail -1 /etc/apache2/httpd.conf
Include /private/etc/apache2/other/*.conf
```

* We then create a symbolic link in */etc/apache2/other* to our newly-copied *httpd-vhosts.conf* from our *~/Sites* folder:

```bash
sudo ln -s ~/Sites/httpd-vhosts.conf /etc/apache2/other
```

* The *~/Sites/httpd-vhosts.conf* is now where we will add all new virtual hosts. We will place our site root folders in *~/Sites* and define them in this file. But first, we need to make some changes (or, [download this edited httpd-vhosts.conf file](http://www.echoditto.com/sites/default/files/labs/httpd-vhosts.conf_.txt) and change **fname** in **/Users/fname** to the appropriate path for your system). Begin by adding this text to the top of the file, which will enable the *~/Sites* folder and subfolders to be accessed by Apache (again, replace */Users/fname* with your appropriate path for home directory):

```
# Ensure all of the VirtualHost directories are listed below
<Directory "/Users/fname/Sites">
    Options Indexes FollowSymLinks MultiViews
    AllowOverride All
    Order allow,deny
    Allow from all
</Directory>
```

* Add these lines anywhere in the file to ensure that http://localhost still serves the default folder:

```
# For localhost in the OS X default location
<VirtualHost _default_:80>
  ServerName localhost
  DocumentRoot /Library/WebServer/Documents
</VirtualHost></code>
```

* We will cover setting up a Virtual Host later, so let's reload our Apache settings to activate our new configuration:

```bash
sudo apachectl graceful
```

Or, go to the **Sharing Preference Pane** and uncheck/[re]check the **Web Sharing** item to stop and start Apache.

![Sharing](http://www.echoditto.com/sites/default/files/1Sharing-1.jpg)

## PHP ##

* As of OS X 10.6.6, OS X ships with PHP 5.3.3, but it is not enabled in Apache by default. By looking at */etc/apache2/httpd.conf*, you can see that the line to load in the PHP module is commented out:

```bash
$ grep php /etc/apache2/httpd.conf
#LoadModule php5_module        libexec/apache2/libphp5.so</code>
```

* Since we are trying to avoid editing default configuration files where possible, we can add this information, along with the configuration to allow *.php files to run, without the comment at the beginning of the line, in the same folder that we put the symlink for *httpd-vhosts.conf*:

```bash
$ sudo sh -c "cat > /etc/apache2/other/php5-loadmodule.conf <<'EOF'
LoadModule php5_module        libexec/apache2/libphp5.so
EOF"
```

* PHP on OS X does not come with a configuration file at php.ini and PHP runs on its defaults. There is, however, a sample file ready to be copied and edited. The lines below will copy the sample file to */etc/php.ini* (optionally, [download the edited file](http://www.echoditto.com/sites/default/files/labs/php.ini_.txt) and insert at */etc/php.ini*) and add some developer-friendly changes to your configuration.

```bash
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
; Original - log_errors = off
log_errors = on
; Original - display_errors = Off
display_errors = on
; Original - display_startup_errors = Off
display_startup_errors = on
; Original - error_reporting = E_ALL & ~E_DEPRECATED
error_reporting = E_ALL & ~E_DEPRECATED & ~E_NOTICE
EOF"
```

* OS X's PHP does ship with pear and pecl, but they are not in the $PATH, nor are there binaries to add or symlink to your path. The first two commands will perform a channel-update for pear and again for pecl, the second two will add an alias for each for your current session, and the last two will add the aliases permanently to your local environment (you may get PHP Notices that appear to be errors but they are simply notices. You can change error_reporting in *php.ini* to silence the Notices):

```bash
$ sudo php /usr/lib/php/pearcmd.php channel-update pear.php.net
$ sudo php /usr/lib/php/peclcmd.php channel-update pecl.php.net
$ alias pear="php /usr/lib/php/pearcmd.php"
$ alias pecl="php /usr/lib/php/peclcmd.php"
$ cat >> ~/.bashrc <<'EOF'

alias pear="php /usr/lib/php/pearcmd.php"
alias pecl="php /usr/lib/php/peclcmd.php"
EOF
```

* We do a lot of Drupal development, and a commonly-added PHP PECL module for Drupal is uploadprogress. With the new alias we just set, adding it is easy. Build the PELC package, add it *php.ini*, and reload the PHP configuration into Apache:
<sub>Note: PECL modules require a compiler such as gcc to build. Install Xcode to build PECL packages such as uploadprogress.</sub>

```bash
$ pecl install uploadprogress
$ sudo sh -c "cat >> /etc/php.ini <<'EOF'

; Enable PECL extension uploadprogress
extension=uploadprogress.so
EOF"
$ sudo apachectl graceful
```

* If you'd like to install APC, see [http://pecl.php.net/bugs/bug.php?id=15968](http://pecl.php.net/bugs/bug.php?id=15968) and search for \*\*[2009-09-10 23:16 UTC]\*\*, or if you are using [Homebrew](http://mxcl.github.com/homebrew/), see [http://i.justrealized.com/2010/install-php-apc-mac-os/](http://i.justrealized.com/2010/install-php-apc-mac-os/)</a>.

## MySQL ##

MySQL is the only thing that doesn't not ship with OS X that is needed to run a "LAMP" website like Drupal or WordPress. MySQL/Sun/Oracle provides pre-built MySQL binaries for OS X that don't conflict with other system packages, and even provides a nice System Preferences pane for starting/stopping the process and a checkbox to start MySQL at boot.

* Go to the download site for OS X pre-built binaries at: [http://dev.mysql.com/downloads/mysql/index.html#macosx-dmg](http://dev.mysql.com/downloads/mysql/index.html#macosx-dmg) and choose the most appropriate type for your system. For systems running Snow Leopard, choose **Mac OS X ver. 10.6 (x86, 64-bit), DMG Archive** (on the following screen, you can scroll down for the download links to avoid having to create a user and password).

![Download DMG](http://www.echoditto.com/sites/default/files/labs/2MySQL%20__%20Download%20MySQL%20Community%20Server.jpg)

* Open the downloaded disk image and begin by installing the **mysql-5.1.51-osx10.6-x86_64.pkg** file (or named similarly to the version number you downloaded):

![MySQL Installer](http://www.echoditto.com/sites/default/files/labs/3Install%20MySQL%205.1.51-community%20for%20Mac%20OS%20X.jpg)

Once the Installer script is completed, install the Preference Pane by double-clicking on **MySQL.prefPane**. If you are not sure where to install the preference pane, choose "Install for this user only":

<sub>**Note:** If you wish to have MySQL start on boot, check out the [first answer on this Stack Overflow post](http://stackoverflow.com/questions/1334272/cant-start-mysql-in-mac-os-10-6-snow-leopard).</sub>

![Install MySQL System Preferences pane](http://www.echoditto.com/sites/default/files/labs/4System%20Preferences.jpg)

* The preference pane is now installed and you can check the box to have MySQL start on boot or start and stop it on demand with this button. To access this screen again, load System Preferences. Click "Start MySQL Server" now:

![MySQL pane](http://www.echoditto.com/sites/default/files/5MySQL.jpg)

* The MySQL installer installed its files to /usr/local/mysql, and the binaries are in /usr/local/mysql/bin which is not in the $PATH of any user by default. Rather than edit the $PATH shell variable, we add symbolic links in /usr/local/bin (a location already in $PATH) to a few MySQL binaries. Add more or omit as desired. This is an optional step, but will make tasks like using Drush or performing a mysqldump in the command line much easier:

```bash
cd /usr/local/bin
ln -s /usr/local/mysql/bin/mysql
ln -s /usr/local/mysql/bin/mysqladmin
ln -s /usr/local/mysql/bin/mysqldump
ln -s /usr/local/mysql/support-files/mysql.server
```

* Next we'll run `mysql_secure_installation` to set the root user's password and other setup-related tasks. This script ships with MySQL and only needs to be run once, and it should be run with sudo. As you run it, you can accept all other defaults after you set the root user's password:

```bash
sudo /usr/local/mysql/bin/mysql_secure_installation
```

* If you were create a simple php file containing "phpinfo();" to view the PHP Information, you will see that Apple-built PHP looks for the MySQL socket at */var/mysql/mysql.sock*:

![phpinfo](http://www.echoditto.com/sites/default/files/labs/6phpinfo.jpg)

The problem is that the MySQL binary provided by Oracle puts the socket file in */tmp*, which is the standard location on most systems. Other tutorials have recommended rebuilding PHP from source, or changing where PHP looks for the socket file via */etc/php.ini*, or changing where MySQL places it when starting via */etc/my.cnf*, but it is far easier and more fool-proof to create a symbolic link for */tmp/mysql.sock* in the location OS X's PHP is looking for it. Since this keeps us from changing defaults, it ensures that a PHP update (via Apple) or a MySQL update (via Oracle) will not have an impact on functionality:

```bash
sudo mkdir /var/mysql
sudo chown _mysql:wheel /var/mysql
sudo ln -s /tmp/mysql.sock /var/mysql
```

* After installing MySQL, several sample my.cnf files are created but none is placed in */etc/my.cnf*, meaning that MySQL will always load in the default configuration. Since we will be using MySQL for local development, we will start with a "small" configuration file and make a few changes to increase the max_allowed_packet variable under **[mysqld]** and **[mysqldump]** (optionally, use the contents of [this sample file](http://www.echoditto.com/sites/default/files/labs/my.cnf_.txt) in */etc/my.cnf*). Later on, we can edit */etc/my.cnf* to make other changes as desired:

```bash
sudo cp /usr/local/mysql/support-files/my-small.cnf /etc/my.cnf
sudo sed -i "" 's/max_allowed_packet = 16M/max_allowed_packet = 2G/g' /etc/my.cnf
sudo sed -i "" "/\[mysqld\]/ a\\`echo -e '\n\r'`max_allowed_packet = 2G`echo -e '\n\r'`" /etc/my.cnf
```

* To load in these changes, go back to the MySQL System Preferences pane, and restart the server by pressing "Stop MySQL Server" followed by "Start MySQL Server." Or, you can do this on the command line (if you added this symlink as shown above):

```
$ sudo mysql.server restart

Shutting down MySQL
... SUCCESS! 
Starting MySQL
... SUCCESS!
```

## phpMyAdmin ##

At this point, the full "MAMP stack" (Macintosh OS X, Apache, MySQL, PHP) is ready, but adding phpMyAdmin will make administering your MySQL install easier.

*   phpMyAdmin will complain that the PHP extension 'php-mcrypt' is not available. Since you will not likely be allowing anyone other than yourself to use phpMyAdmin, you could ignore it, but if you want to build the extension, it is possible to build it as a shared object instead of having to rebuild PHP from source. Follow Michael Gracie's guide here (for his Step 2, grab the PHP source that matches `$ php --version` on your system): [http://michaelgracie.com/2009/09/23/plugging-mcrypt-into-php-on-mac-os-x-snow-leopard-10.6.1/](http://michaelgracie.com/2009/09/23/plugging-mcrypt-into-php-on-mac-os-x-snow-leopard-10.6.1/) (or, if you prefer Homebrew: [http://blog.rogeriopvl.com/archives/php-mcrypt-in-snow-leopard-with-homebrew/](http://blog.rogeriopvl.com/archives/php-mcrypt-in-snow-leopard-with-homebrew/).

    The following files will be added:
    *    /usr/local/lib/libmcrypt
    *    /usr/local/include/mcrypt.h
    *    /usr/local/include/mutils/mcrypt.h
    *    /usr/local/bin/libmcrypt-config
    *    /usr/local/lib/libmcrypt.la
    *    /usr/local/lib/libmcrypt.4.4.8.dylib
    *    /usr/local/lib/libmcrypt.4.dylib
    *    /usr/local/lib/libmcrypt.dylib
    *    /usr/local/share/aclocal/libmcrypt.m4
    *    /usr/local/man/man3/mcrypt.3
    *    /usr/lib/php/extensions/no-debug-non-zts-20090626/mcrypt.so

*   The following lines will install the latest phpMyAdmin to */Library/WebServer/Documents* and set up *config.inc.php*. Afterwards, you will be able to access phpMyAdmin at <a href="http://localhost/phpMyAdmin">http://localhost/phpMyAdmin</a>:

```bash
cd /Library/WebServer/Documents
curl -s -L -O http://downloads.sourceforge.net/project/phpmyadmin/phpMyAdmin/`curl -s -L http://www.phpmyadmin.net/home_page/index.php | grep -m1 dlname |awk -F'Download ' '{print $2}'|cut -d "<" -f1`/phpMyAdmin-`curl -s -L http://www.phpmyadmin.net/home_page/index.php | grep -m1 dlname |awk -F'Download ' '{print $2}'|cut -d "<" -f1`-english.tar.gz
tar zxpf `ls -1 phpMyAdmin*tar.gz|sort|tail -1`
rm `ls -1 phpMyAdmin*tar.gz|sort|tail -1`
mv `ls -d1 phpMyAdmin-*|sort|tail -1` phpMyAdmin
cd phpMyAdmin
cp config.sample.inc.php config.inc.php
sed -i "" "s/blowfish_secret\'\] = \'/blowfish_secret\'\] = \'`cat /dev/urandom | strings | grep -o '[[:alnum:]]' | head -n 50 | tr -d '\n'`/" config.inc.php
```

## Drush ##

Drush is a helpful tool that provides a "Drupal shell" to speed up your Drupal development.

*   Run the following in the Terminal to get Drush 4.2 installed in */usr/local* and add drush to the $PATH:

```bash
svn co http://subversible.svn.beanstalkapp.com/modules/drush/tags/DRUPAL-7--4-2/ /usr/local/drush
cd /tmp
pear download Console_Table
tar zxvf `ls Console_Table-*.tgz` `ls Console_Table-*.tgz | sed -e 's/\.[a-z]\{3\}$//'`/Table.php
mv `ls Console_Table-*.tgz | sed -e 's/\.[a-z]\{3\}$//'`/Table.php /usr/local/drush/includes/table.inc
rm -fr Console_Table-*
ln -s /usr/local/drush/drush /usr/local/bin/drush</code>
```

## Virtual Hosts ##

Now that all of the components to run a website locally are in place, it only takes a few changes to *~/Sites/httpd-vhosts.conf* to get a new local site up and running.

*   Your website code should be in a directory in *~/Sites*. For the purposes of this example, the webroot will be at *~/Sites/thewebsite*.

*   You need to choose a "domain name" for your local site that you will use to access it in your browser. For the example, we will use "myproject.local". The first thing you need to do is add a line to */etc/hosts* to direct your "domain name" to your local system:

```
$ sudo sh -c "cat >> /etc/hosts <<'EOF'
127.0.0.1 myproject.local
EOF
```

* If you have been following the rest of this guide and used a copy of */etc/apache2/extra/httpd-vhosts.conf* in *~/Sites/httpd-vhosts.conf*, you can delete the first of the two example Virtual Hosts and edit the second one. You can also delete the **ServerAdmin** line as it is not necessary.

* Change the **ServerName** to **myproject.local** (or whatever you entered into */etc/hosts*) so Apache will respond when you visit [http://myproject.local](http://myproject.local) in a browser.

* Change the **DocumentRoot** to the path of your webroot, which in this example is **/Users/fname/Sites/thewebsite**

* It's highly recommended to set a CustomLog and an ErrorLog for each virtual host, otherwise all logged output will be in the same file for all sites. Change the line starting with **CustomLog** from "dummy-host2.example.com-access_log" to "myproject.local-access_log", and for **ErrorLog** change "dummy-host2.example.com-error_log" to "myproject.local-error_log"

* Save the file and reload the Apache configuration as shown in Step 6 under the Apache section, and visit [http://myproject.local](http://myproject.local) in your browser.

Going forward, all you have to do to add a new website is duplicate and edit the **\<VirtualHost *:80\>** to **\</VirtualHost\>** section underneath your existing configuration, make new edits, edit */etc/hosts* again, and reload Apache.