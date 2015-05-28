---
layout: post
title: "OS X 10.9 Local Development Environment: Apache, PHP, and MySQL with Homebrew"
date: 2014-06-09 09:55
comments: true
categories: 
---

*This post originally featured on the [Echo &amp; Co. blog](http://echo.co/blog/os-x-109-local-development-environment-apache-php-and-mysql-homebrew)*.

<style>code {display:inline;padding:0;margin:0;border:none;}</style>
There's nothing quite like setting up everything on your Mac for Drupal (or other PHP) development in a way that things just work and don't need constant fiddling. This guide will walk you through using [Homebrew](http://brew.sh/) to install Apache, PHP, and MySQL for a "MAMP" development environment. We'll also use DNSMasq and Apache's VirtualDocumentRoot to set up "auto-VirtualHosts," and add a firewall rule to allow the default http port 80 to be used without running Apache as root.

At the conclusion of this guide, you'll be able to create a directory like *~/Sites/project* and access it at *http://project.dev* without editing your */etc/hosts* file or editing any Apache configuration. You'll also be able to use [xip.io](http://xip.io/) with auto-VirtualHosts for accessing your sites on other devices in your local network.

We also configure PHP and MySQL to allow for enough flexibility for complex operations generally only reserved for development and not production.

The OS X operating system comes with Apache and PHP pre-installed, and I've [previously](/blog/os-x-107-lion-development-native-apache-php-homebrew-mysql-or-mariadb) [recommended](/blog/os-x-107-lion-development-native-mamp-mysql-installer) utilizing them to some degree for getting a local PHP development environment on your Mac. Since then, the community around Homebrew has improved dramatically and I now recommend that our developers at Echo & Co. use Homebrew exclusively for all components.

# Homebrew Setup

If you've not already install Homebrew, you can follow the instructions at [brew.sh](http://brew.sh/), or simply run the following command:

<pre language="bash">
ruby -e "$(curl -fsSL https://raw.github.com/Homebrew/homebrew/go/install)"
brew doctor
brew update
</pre>

Also of note, if you do not have `git` available on your system, either from Homebrew, Xcode, or another source, you can install it with Homebrew now (if you already have it installed, feel free to skip this step to keep the version of `git` you already have):

<pre language="bash">
brew install -v git
</pre>

# PATH Variable

Since OS X already comes with PHP and Apache, we'll want to make sure that our `brew`-installed versions appear in the shell path before the built-in ones. The following command adds logic to your *~/.bash_profile* to ensure the Homebrew directory is in the beginning of **$PATH**.

<pre language="bash">
echo "export PATH=\$(echo \$PATH | sed 's|/usr/local/bin||; s|/usr/local/sbin||; s|::|:|; s|^:||; s|\(.*\)|/usr/local/bin:/usr/local/sbin:\1|')" >> ~/.bash_profile && source ~/.bash_profile
</pre>


# MySQL

The following commands will download and install the latest version of MySQL and do some basic configuration to allow for large imports and a couple other miscellaneous configuration changes. 

<pre language="bash">
brew install -v mysql

cp -v $(brew --prefix mysql)/support-files/my-default.cnf $(brew --prefix mysql)/my.cnf

cat >> $(brew --prefix mysql)/my.cnf <<'EOF'
# Echo & Co. changes
max_allowed_packet = 2G
innodb_file_per_table = 1
EOF

sed -i '' 's/^# \(innodb_buffer_pool_size\)/\1/' $(brew --prefix mysql)/my.cnf
</pre>

Now we need to start MySQL using OS X's launchd, and we'll set it to start when you login.

<pre language="bash">
[ ! -d ~/Library/LaunchAgents ] && mkdir -v ~/Library/LaunchAgents

[ -f $(brew --prefix mysql)/homebrew.mxcl.mysql.plist ] && ln -sfv $(brew --prefix mysql)/homebrew.mxcl.mysql.plist ~/Library/LaunchAgents/

[ -e ~/Library/LaunchAgents/homebrew.mxcl.mysql.plist ] && launchctl load -w ~/Library/LaunchAgents/homebrew.mxcl.mysql.plist
</pre>

By default, MySQL's root user has an empty password from any connection. You are advised to run `mysql_secure_installation` and at least set a password for the root user. 

<pre language="bash">
$(brew --prefix mysql)/bin/mysql_secure_installation
</pre>

# Apache

Start by stopping the built-in Apache, if it's running, and prevent it from starting on boot. This is one of very few times you'll need to use `sudo`.

<pre language="bash">
sudo launchctl unload -w /System/Library/LaunchDaemons/org.apache.httpd.plist 2>/dev/null
</pre>

Apache is not in the default repository of Homebrew formulas, nor are some dependencies, so we use the `brew tap` command to add other formula repositories.

<pre language="bash">
brew tap homebrew/dupes
brew tap homebrew/apache
</pre>

We'll install Apache 2.2 with Apr and OpenSSL from Homebrew as well instead of utilizing the built-in versions of those tools. 

<pre language="bash">
brew install -v httpd22 --with-brewed-openssl
</pre>

We'll be using the file *~/Sites/httpd-vhosts.conf* to configure our VirtualHosts, so we create necessary directories and then include the file in *httpd.conf*.

<pre language="bash">
[ ! -d ~/Sites ] && mkdir -pv ~/Sites

touch ~/Sites/httpd-vhosts.conf

USERHOME=$(dscl . -read /Users/`whoami` NFSHomeDirectory | awk -F": " '{print $2}') cat >> $(brew --prefix)/etc/apache2/2.2/httpd.conf <<EOF
# Include our VirtualHosts
Include ${USERHOME}/Sites/httpd-vhosts.conf
EOF
</pre>

We'll create a folder for our logs in *~/Sites* as well.

<pre language="bash">
[ ! -d ~/Sites/logs ] && mkdir -pv ~/Sites/logs
</pre>

Now to fill in the contents of *~/Sites/httpd-vhosts.conf* that we included in *httpd.conf* earlier. Note that **this is one command to copy and paste!** Start with `USERHOME` through the second `EOF` has a single copy and paste block for the terminal.

<pre language="bash">
USERHOME=$(dscl . -read /Users/`whoami` NFSHomeDirectory | awk -F": " '{print $2}') cat > ~/Sites/httpd-vhosts.conf <<EOF
#
# Use name-based virtual hosting.
#
NameVirtualHost *:80

#
# Set up permissions for VirtualHosts in ~/Sites
#
<Directory "${USERHOME}/Sites">
    Options Indexes FollowSymLinks MultiViews
    AllowOverride All
    Order allow,deny
    Allow from all
</Directory>

# For http://localhost in the users' Sites folder
<VirtualHost _default_:80>
    ServerName localhost
    DocumentRoot "${USERHOME}/Sites"
</VirtualHost>

#
# VirtualHosts
#

## Manual VirtualHost template
#<VirtualHost *:80>
#  ServerName project.dev
#  CustomLog "${USERHOME}/Sites/logs/project.dev-access_log" combined
#  ErrorLog "${USERHOME}/Sites/logs/project.dev-error_log"
#  DocumentRoot "${USERHOME}/Sites/project.dev"
#</VirtualHost>

#
# Automatic VirtualHosts
# A directory at ${USERHOME}/Sites/webroot can be accessed at http://webroot.dev
# In Drupal, uncomment the line with: RewriteBase /

# This log format will display the per-virtual-host as the first field followed by a typical log line
LogFormat "%V %h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combinedmassvhost

# Auto-VirtualHosts with .dev
<VirtualHost *:80>
  ServerName dev
  ServerAlias *.dev

  CustomLog "${USERHOME}/Sites/logs/dev-access_log" combinedmassvhost
  ErrorLog "${USERHOME}/Sites/logs/dev-error_log"

  VirtualDocumentRoot ${USERHOME}/Sites/%-2+
</VirtualHost>

# Auto-VirtualHosts with xip.io
<VirtualHost *:80>
  ServerName xip
  ServerAlias *.xip.io

  CustomLog "${USERHOME}/Sites/logs/dev-access_log" combinedmassvhost
  ErrorLog "${USERHOME}/Sites/logs/dev-error_log"

  VirtualDocumentRoot ${USERHOME}/Sites/%-7+
</VirtualHost>
EOF
</pre>

## Run with port 80

You may notice that *httpd.conf* is running Apache on port **8080**, but the <VirtualHosts> above are using port **80**. The next two commands will create and load a firewall rule to forward 8080 requests to 80. The end result is that we can use port 80 in VirtualHosts without needing to run Apache as root.

The following **single** command will create the file */Library/LaunchDaemons/co.echo.httpdfwd.plist* as root, and owned by root, since it needs elevated privileges.

<pre language="bash">
sudo bash -c 'export TAB=$'"'"'\t'"'"'
cat > /Library/LaunchDaemons/co.echo.httpdfwd.plist <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
${TAB}<key>Label</key>
${TAB}<string>co.echo.httpdfwd</string>
${TAB}<key>ProgramArguments</key>
${TAB}<array>
${TAB}${TAB}<string>sh</string>
${TAB}${TAB}<string>-c</string>
${TAB}${TAB}<string>ipfw add fwd 127.0.0.1,8080 tcp from any to me dst-port 80 in &amp;&amp; sysctl -w net.inet.ip.forwarding=1</string>
${TAB}</array>
${TAB}<key>RunAtLoad</key>
${TAB}<true/>
${TAB}<key>UserName</key>
${TAB}<string>root</string>
</dict>
</plist>
EOF'
</pre>

This file will be loaded on login and set up the 80->8080 port forward, but we can load it manually now so we don't need to log out and back in.

<pre language="bash">
sudo launchctl load -w /Library/LaunchDaemons/co.echo.httpdfwd.plist
</pre>

# PHP

The following is for the latest release of PHP, version 5.5. If you'd like to use 5.3, 5.4 or 5.6, simply change the "5.5" and "php55" values below appropriately. *(Note: if you use 5.3, the OpCache extension instructions are different. They will be posted below after the instructions for newer versions.)*

Start by adding the PHP tap for Homebrew. PHP 5.3 needs an additional tap, so skip the second command if you are using 5.4 or higher.

<pre language="bash">
brew tap homebrew/php

# Skip this if using PHP 5.4 or higher
brew tap homebrew/versions
</pre>

Install PHP and mod_php. This command will also load the PHP module in the *httpd.conf* file for you.

<pre language="bash">
brew install -v php55 --homebrew-apxs --with-apache
</pre>

Add PHP configuration to Apache's *httpd.conf* file.

<pre language="bash">
cat >> $(brew --prefix)/etc/apache2/2.2/httpd.conf <<EOF
# Send PHP extensions to mod_php
AddHandler php5-script .php
AddType text/html .php
DirectoryIndex index.php index.html
EOF
</pre>

Set timezone and change other PHP settings (this is a **single** command). `sudo` is needed here to get the current timezone on OS X (in previous versions of OS X it wasn't needed, I'm not sure why it is now).

<pre language="bash">
sed -i '-default' "s|^;\(date\.timezone[[:space:]]*=\).*|\1 \"$(sudo systemsetup -gettimezone|awk -F": " '{print $2}')\"|; s|^\(memory_limit[[:space:]]*=\).*|\1 256M|; s|^\(post_max_size[[:space:]]*=\).*|\1 200M|; s|^\(upload_max_filesize[[:space:]]*=\).*|\1 100M|; s|^\(default_socket_timeout[[:space:]]*=\).*|\1 600|; s|^\(max_execution_time[[:space:]]*=\).*|\1 300|; s|^\(max_input_time[[:space:]]*=\).*|\1 600|;" $(brew --prefix)/etc/php/5.5/php.ini
</pre>

Add a PHP error log; without this, you may get Internal Server Errors if PHP has errors to write and no logs to write to (this is a **single** command; be sure to copy and paste the lines containing `USERHOME` through the last `EOF` as a **single** command).

<pre language="bash">
USERHOME=$(dscl . -read /Users/`whoami` NFSHomeDirectory | awk -F": " '{print $2}') cat >> $(brew --prefix)/etc/php/5.5/php.ini <<EOF
; PHP Error log
error_log = ${USERHOME}/Sites/logs/php-error_log
EOF
</pre>

This weird little "hack" is needed to fix [a permissions problem](https://github.com/Homebrew/homebrew-php/issues/1039#issuecomment-41307694) with using `pear` or `pecl`.

<pre language="bash">
touch $(brew --prefix php55)/lib/php/.lock && chmod 0644 $(brew --prefix php55)/lib/php/.lock
</pre>

The OpCache extension will speed up your PHP environment dramatically, and it's easy to install for 5.4 and higher. **If you are looking to install PHP 5.3, skip this block.**

<pre language="bash">
brew install -v php55-opcache
</pre>

**Skip this block unless you are using PHP 5.3**. Because there is no *php53-opcache* Homebrew formula, we can install it with `pecl` and replicate the same configuration file.

<pre language="bash">
pecl install zendopcache-beta

sed -i '' "s|^\(zend_extension=\"\)\(opcache\.so\"\)|\1$(php -r 'print(ini_get("extension_dir")."/");')\2|" $(brew --prefix)/etc/php/5.3/php.ini

[ ! -d $(brew --prefix)/etc/php/5.3/conf.d ] && mkdir -pv $(brew --prefix)/etc/php/5.3/conf.d

echo "[opcache]" > $(brew --prefix)/etc/php/5.3/conf.d/ext-opcache.ini

grep -E '^zend_extension.*opcache\.so' $(brew --prefix)/etc/php/5.3/php.ini >> $(brew --prefix)/etc/php/5.3/conf.d/ext-opcache.ini

sed -i '' '/^zend_extension.*opcache\.so/d' $(brew --prefix)/etc/php/5.3/php.ini

# "php54" is not a typo here- I'm using a sample config file from
# another recipe for my config file in php53
grep -E '^[[:space:]]*opcache\.' \
  $(brew --prefix)/Library/Taps/homebrew/homebrew-php/Formula/php54-opcache.rb \
  | sed 's/^[[:space:]]*//g' >> $(brew --prefix)/etc/php/5.3/conf.d/ext-opcache.ini
</pre>

And continue with steps for all PHP versions: give OpCache some more memory to keep more opcode caches.

<pre language="bash">
sed -i '' "s|^\(opcache\.memory_consumption=\)[0-9]*|\1256|;" $(brew --prefix)/etc/php/5.5/conf.d/ext-opcache.ini
</pre>

# Start Apache

Start Homebrew's Apache and set to start on boot.

<pre language="bash">
ln -sfv $(brew --prefix httpd22)/homebrew.mxcl.httpd22.plist ~/Library/LaunchAgents

launchctl load -w ~/Library/LaunchAgents/homebrew.mxcl.httpd22.plist
</pre>

# DNSMasq

[I've covered this before](/blog/never-touch-your-local-etchosts-file-os-x-again), but we'll list the commands here again for completeness. This example will have any DNS request ending in *.dev* reply with the IP address *127.0.0.1*.

<pre language="bash">
brew install -v dnsmasq

echo 'address=/.dev/127.0.0.1' > $(brew --prefix)/etc/dnsmasq.conf

echo 'listen-address=127.0.0.1' >> $(brew --prefix)/etc/dnsmasq.conf
</pre>

Because DNS services run on a lower port, we need to have this run out of */Library/LaunchDaemons*, so we do need to use `sudo` for the initial setup.

<pre language="bash">
sudo cp -v $(brew --prefix dnsmasq)/homebrew.mxcl.dnsmasq.plist /Library/LaunchDaemons 

sudo launchctl load -w /Library/LaunchDaemons/homebrew.mxcl.dnsmasq.plist 
</pre>

With DNSMasq running, configure OS X to use your local host for DNS queries ending in *.dev*

<pre language="bash">
sudo mkdir -v /etc/resolver 

sudo bash -c 'echo "nameserver 127.0.0.1" > /etc/resolver/dev'
</pre>

# Great! So, what did I do?

We set up Apache to run on boot on port 8080 with mod_php with auto-VirtualHosts for directories in the *~/Sites* folder. The OS X firewall will forward all port 80 traffic to port 8080, so we don't have specify the port number when visiting web pages in web browsers. MySQL is installed and set to run on boot as well. DNSMasq and some OS X configuration is used to direct any hostname ending in *.dev* to the local system to work in conjunction with Apache's auto-VirtualHosts.

## What do I do now?

You shouldn't need to edit the Apache configuration or edit /etc/hosts for new local development sites. Simply create a directory in *~/Sites* and then reference that foldername + .dev in your browser to access it. 

For example, use `drush` to download Drupal 7 to the directory *~/Sites/firstproject*, and it can then be accessed at *http://firstproject.dev/* without any additional configuration. A caveat - you will need to uncomment the line in Drupal's .htaccess containing **"RewriteBase /"** to work with the auto-VirtualHosts configuration. 

## What about this xip.io thing?

If your Mac's LAN IP address is *192.168.0.10*, you can access sites from any other device on your local network using *http://firstproject.192.168.0.10.xip.io/*. You can test a websites on your Mac with your phone/tablet/etc. Pretty great, right?

## What if this "auto-VirtualHost" doesn't work for me?

If you need to create a manual VirtualHost in Apache because the auto-VirtualHost does not work for your configuration, you may need to declare it before the auto-VirtualHosts if you're also going to use .dev as the TLD. Otherwise the auto-VirtualHost block will be accessed first.