---
layout: post
title: "OS X 10.7 Lion Development: Native MAMP with MySQL installer"
date: 2011-08-24 15:30
comments: true
categories:
---

*This post originally featured on the [Echo &amp; Co. blog](http://echo.co/blog/os-x-107-lion-development-native-mamp-mysql-installer)*.

With the release of Lion, there are some subtle differences to setting up a local MAMP (Mac OS X, Apache, MySQL, PHP) environment compared to Snow Leopard. In an effort to keep this from being overly wordy and just get to the good stuff, we'll dive right in, so read on to get started.

Note that for all commands before that are starting with a $, the dollar sign is showing a command-line prompt in [Terminal](http://www.apple.com/macosx/apps/all.html#terminal), and you should not actually type it as part of the commands.

## Apache

We'll set things up so we won't need sudo often in the future, and so we can manage multiple VirtualHosts. We'll keep the Apache information and our website roots in ~/Sites, and Apache logs in ~/Sites/logs

```bash
$ [ ! -d ~/Sites ] && mkdir ~/Sites 
$ touch ~/Sites/httpd-vhosts.conf 
$ sudo ln -s ~/Sites/httpd-vhosts.conf /etc/apache2/other 
$ mkdir ~/Sites/logs 
$ chmod 0777 ~/Sites/logs
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
$ sudo bash -c "grep php /etc/apache2/httpd.conf|grep LoadModule|cut -d'#' -f2 > /etc/apache2/other/php5-loadmodule.conf" 
$ sudo cp -a /etc/php.ini.default /etc/php.ini 
$ sudo bash -c "cat >> /etc/php.ini <<'EOF' 


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

**Optional:** Lion ships with PEAR but not installed, whereas Snow Leopard it was installed. This will install PEAR from the phar archive, upgrade it, and add `pear` and `pecl` aliases to your shell.

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

Note that installing most pecl modules will require having a compiler installed via Xcode. I'll be addressing installing [uploadprogress](http://pecl.php.net/package/uploadprogress) and [APC](http://pecl.php.net/package/apc) in a later post, since this walkthrough requires no compiling.

Toggle **Web Sharing** in **System Preferences &gt; Sharing** for the new PHP options to take effect.

## MySQL

MySQL is the only thing not shipped with OS X that we need for our development environment. Go to the download site for OS X pre-built binaries at http://dev.mysql.com/downloads/mysql/index.html#macosx-dmg and choose the **DMG Archive** most appropriate for your system. As of this writing, the most recent version says *OS X 10.6*, but it will work on 10.7 as well.

Open the downloaded disk image and begin by installing the *mysql-5.x.x-osx10.6-x86_64.pkg* file. Once the Installer script is completed, install the Preference Pane by double-clicking on MySQL.prefPane. If you are not sure where to install the preference pane, choose "Install for this user only."

**Optional:** If you wish to have MySQL start on boot, you do need to additionally install the MySQLStartupItem.pkg file. The preference pane has a toggle for starting on boot, but it will not work until you install the pkg and you run the following in Terminal:

```bash
sudo chown -R root:wheel /Library/StartupItems/MySQLCOM
```

Open **System Preferences** and **MySQL** and start the database by clicking the **Start MySQL Server** button.

**Optional:** The MySQL installer installed its files to /usr/local/mysql, and the binaries are in /usr/local/mysql/bin which is not in the $PATH of any user by default. Rather than edit the $PATH shell variable, we add symbolic links in /usr/local/bin (a location already in $PATH) to a few MySQL binaries. Add more or omit more binaries as desired:

```bash
[ ! -d /usr/local/bin ] && sudo mkdir -p /usr/local && sudo chmod 0777 /usr/local && mkdir /usr/local/bin && sudo chmod 0755 /usr/local 
cd /usr/local/bin 
ln -s /usr/local/mysql/bin/mysql 
ln -s /usr/local/mysql/bin/mysqladmin 
ln -s /usr/local/mysql/bin/mysqldump 
ln -s /usr/local/mysql/support-files/mysql.server
```

This script ships with MySQL and only needs to be run once, and it should be run with sudo. As you run it, you can accept all other defaults after you set the root user's password:

```bash
sudo /usr/local/mysql/bin/mysql_secure_installation
```

After installing MySQL, several sample my.cnf files are created but none is placed where MySQL can find it. Start with a "small" configuration file and make a few changes to increase the max_allowed_packet variable:

```bash
sudo cp /usr/local/mysql/support-files/my-small.cnf /usr/local/mysql/data/my.cnf 
sudo sed -i "" 's/max_allowed_packet = 1.*M/max_allowed_packet = 2G/g' /usr/local/mysql/data/my.cnf
```

To load in these changes, go back to the MySQL System Preferences pane, and restart the server by pressing "Stop MySQL Server" followed by "Start MySQL Server."

## Make PHP and MySQL Play Nice

If you were to run `php -i|egrep 'mysql.*default_socket'` you would see that PHP was compiled to expect the MySQL socket file in /var/mysql, but MySQL will place it in /tmp. This easiest fix is to tell PHP to look in /tmp:

```bash
sudo sed -i "" 's/\/var\/mysql\/mysql\.sock/\/tmp\/mysql\.sock/g' /etc/php.ini
```

Toggle **Web Sharing** in **System Preferences &gt; Sharing** for the new PHP options to take effect.

## Hooray!

You should now be all set to keep adding more VirtualHosts in httpd-vhosts.conf and begin development on your local machine. Stay tuned for instructions on how to use [MariaDB](http://mariadb.org/) or MySQL with [Homebrew](http://mxcl.github.com/homebrew/) instead of the Oracle MySQL installer, and [MacPorts](http://www.macports.org/)!