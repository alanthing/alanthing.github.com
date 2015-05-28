---
layout: post
title: "Usable MAMP on OS X 10.8 Mountain Lion"
date: 2012-12-19 11:18
comments: true
categories:
---

*This post originally featured on the [Echo &amp; Co. blog](http://echo.co/blog/usable-mamp-os-x-108-mountain-lion)*.

## Install MAMP

Download the latest MAMP from <http://mamp.info/en/downloads/index.html> and run the installer.

## Choose APC Cache

Open the MAMP app, open **Preferences...**, click the **PHP** tab, and change **Cache** to **APC**. Click **OK** to close Preferences and **Quit** MAMP.

![MAMP Screenshot](http://i46.tinypic.com/2wltqpw.png)

## PATH variable

Put MAMP binaries, including PHP, in the front of your $PATH (this is a single command):

```bash
echo 'export PATH="/Applications/MAMP/bin:/Applications/MAMP/Library/bin:$(find /Applications/MAMP/bin/php -type d -name "php5.4*" | sort | tail -1)/bin:$PATH"' >> ~/.bash_profile && source ~/.bash_profile
```

## MySQL

Set up MySQL and stop it (default MySQL password is **root**, you do not have to change it when running `mysql_secure_installation`, though I recommend all of the other defaults):

```bash
cp -va /Applications/MAMP/Library/support-files/my-small.cnf /Applications/MAMP/conf/my.cnf

sed -i '' "s/^\(max_allowed_packet =\) [0-9]*M/\1 1G/g" /Applications/MAMP/conf/my.cnf

egrep "^# Uncomment the following if you are using InnoDB tables$" /Applications/MAMP/conf/my.cnf &>/dev/null && sed -i '' "s/^\(# Uncomment the following if you are using InnoDB tables\)$/\1@innodb_file_per_table/; y/@/\n/; s/^#\(innodb_.*\)/\1/g" /Applications/MAMP/conf/my.cnf

/Applications/MAMP/bin/startMysql.sh &

# Secure MySQL setup:
/Applications/MAMP/Library/bin/mysql_secure_installation
# -- OR --
# Less secure MySQL setup:
#/Applications/MAMP/Library/bin/mysql -uroot -proot -e"DELETE FROM mysql.user WHERE User=''; DELETE FROM mysql.user WHERE User='root' AND Host!='localhost'; FLUSH PRIVILEGES;"

/Applications/MAMP/bin/stopMysql.sh
```

## PHP

Set timezone, and increase timeouts and memory values:

```bash
sed -i '-default' "s|^;*\(date\.timezone[[:space:]]*=\).*|\1 \"`systemsetup -gettimezone|awk -F"\: " '{print $2}'`\"|; s|^\(memory_limit[[:space:]]*=\).*\(\;.*\)|\1 256M \2|; s|^\(post_max_size[[:space:]]*=\).*|\1 200M|; s|^\(upload_max_filesize[[:space:]]*=\).*|\1 100M|; s|^\(default_socket_timeout[[:space:]]*=\).*|\1 600|; s|^\(max_execution_time[[:space:]]*=\).*\(\;.*\)|\1 300 \2|; s|^\(max_input_time[[:space:]]*=\).*\(\;.*\)|\1 600 \2|;" $(find /Applications/MAMP/bin/php -type d -name "php5.4*" | sort | tail -1)/conf/php.ini

echo -e "\n[apc]\napc.shm_size = 192M\napc.rfc1867 = 1" >> $(find /Applications/MAMP/bin/php -type d -name "php5.4*" | sort | tail -1)/conf/php.ini
```

### PECL Uploadprogress

**Feel free to skip:** This is quite involved when `apc.rfc1867=1` does the job in most cases. This is mostly an exercise in building PHP modules for MAMP. 

**You'll need to install Xcode Command Line Tools from <http://develop.apple.com/downloads> in order to complete all of the steps below.**

```bash
mkdir -p $(find /Applications/MAMP/bin/php -type d -name "php5.4*" | sort | tail -1)/include
cd $(find /Applications/MAMP/bin/php -type d -name "php5.4*" | sort | tail -1)/include
export PHPVER=$(cd /Applications/MAMP/bin/php && find * -type d -name "php5.4*" | sort | tail -1 | sed 's/php//') && curl -L -o php-${PHPVER}.tar.bz2 http://us3.php.net/get/php-${PHPVER}.tar.bz2/from/us3.php.net/mirror && unset PHPVER
ls php-5.4*bz2 | xargs -L1 tar jxpf && rm php-5.4*bz2
mv -v $(find * -type d -name "php-5.4*") php
cd php
./configure
cd ..

## Build automake and related tools
export build="$PWD/build"
mkdir -p $build
cd $build
curl -OL http://ftpmirror.gnu.org/autoconf/autoconf-2.68.tar.gz
tar xzf autoconf-2.68.tar.gz
cd autoconf-2.68
./configure --prefix=$build/autotools-bin
make
make install
export PATH=$PATH:$build/autotools-bin/bin
cd $build
curl -OL http://ftpmirror.gnu.org/automake/automake-1.11.tar.gz
tar xzf automake-1.11.tar.gz
cd automake-1.11
./configure --prefix=$build/autotools-bin
make
make install
cd $build
curl -OL http://ftpmirror.gnu.org/libtool/libtool-2.4.tar.gz
tar xzf libtool-2.4.tar.gz
cd libtool-2.4
./configure --prefix=$build/autotools-bin
make
make install
cd $build
unset build
rm autoconf-2.68.tar.gz automake-1.11.tar.gz libtool-2.4.tar.gz
rm -rf autoconf-2.68 automake-1.11 libtool-2.4
cd ..

pecl download uploadprogress
ls uploadprogress*z | xargs -L1 tar zxpf && rm -v uploadprogress*z package*xml
mv -v $(find * -type d -name "uploadprogress*") uploadprogress
cd uploadprogress
phpize
./configure --with-php-config=$(find /Applications/MAMP/bin/php -type d -name "php5.4*" | sort | tail -1)/bin/php-config
make
make install

echo -e "\n[uploadprogress]\nextension=uploadprogress.so" >> $(find /Applications/MAMP/bin/php -type d -name "php5.4*" | sort | tail -1)/conf/php.ini
```

Reference 1: <http://www.lullabot.com/articles/installing-php-pear-and-pecl-extensions-on-mamp-mac-os-x-107-lion>
Reference 2: <http://jsdelfino.blogspot.com/2012/08/autoconf-and-automake-on-mac-os-x.html>

## Apache

Set up VirtualHosts in **~/Sites/mamp-vhosts.conf** so it's easy to edit later:

```bash
cp -av /Applications/MAMP/conf/apache/httpd.conf{,-default}

export USERHOME=$(dscl . -read /Users/`whoami` NFSHomeDirectory | awk -F"\: " '{print $2}') && echo -e "\n# User VirtualHosts file (added after MAMP installer)\nInclude ${USERHOME}/Sites/mamp-vhosts.conf" >> /Applications/MAMP/conf/apache/httpd.conf && unset USERHOME

[ ! -d ~/Sites/logs ] && mkdir -pv ~/Sites/logs
```

***IMPORTANT!*** Be sure to copy and paste the lines containing `PORTNUM` through the last `EOF` as a ***single*** command (this entire block is a single copy+paste):

```apache
PORTNUM=$(egrep '^Listen [0-9]*' /Applications/MAMP/conf/apache/httpd.conf | awk '{print $2}') USERHOME=$(dscl . -read /Users/`whoami` NFSHomeDirectory | awk -F"\: " '{print $2}') cat > ~/Sites/mamp-vhosts.conf <<EOF
#
# Use name-based virtual hosting.
#
NameVirtualHost *:${PORTNUM}

#
# Set up permissions for VirtualHosts in ~/Sites
#
<Directory "${USERHOME}/Sites">
    Options Indexes FollowSymLinks MultiViews
    AllowOverride All
    Order allow,deny
    Allow from all
</Directory>

# For http://localhost in the MAMP default location
<VirtualHost _default_:${PORTNUM}>
    ServerName localhost
    DocumentRoot "/Applications/MAMP/htdocs"
</VirtualHost>

#
# VirtualHosts
#

## VirtualHost template
#<VirtualHost *:${PORTNUM}>
#  ServerName domain.local
#  CustomLog "${USERHOME}/Sites/logs/domain.local-access_log" combined
#  ErrorLog "${USERHOME}/Sites/logs/domain.local-error_log"
#  DocumentRoot "${USERHOME}/Sites/domain.local"
#</VirtualHost>

#
# Automatic VirtualHosts
# A directory at ${USERHOME}/Sites/webroot can be accessed at http://webroot.dev
# In Drupal, uncomment the line in .htaccess with: RewriteBase /
#
# This log format will display the per-virtual-host as the first field followed by a typical log line
LogFormat "%V %h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combinedmassvhost
<VirtualHost *:${PORTNUM}>
  ServerName dev
  ServerAlias *.dev

  CustomLog "${USERHOME}/Sites/logs/dev-access_log" combinedmassvhost
  ErrorLog "${USERHOME}/Sites/logs/dev-error_log"

  VirtualDocumentRoot ${USERHOME}/Sites/%-2+
</VirtualHost>
EOF
```

If you read that closely, you'll see that it's set up to do [MassVirtualHosts](http://httpd.apache.org/docs/2.2/vhosts/mass.html) using the `*.dev` tld. For example, set up a web root at `$HOME/Sites/project` and you will be able to view it at <http://project.dev:8888> without needing to create a separate VirtualHost. A custom log format is used, putting the domain name at the beginning of the line, so you can easily see the output as it related to a project.

## Finishing Up

Open the MAMP application to **Start Servers**. Note that if you ever change the Apache port number, you'll need to edit `~/Sites/mamp-vhosts.conf` appropriately.

As always with MAMP, you can work out of <http://localhost:8888/> by putting files in `/Applications/MAMP/htdocs`. For example, the folder `/Applications/MAMP/drupal` will be accessible at <http://localhost:8888/drupal>. You can find other MAMP tools at <http://localhost:8888/MAMP>.

If you're having trouble connecting to MySQL with Sequel Pro or another utility, use `/Applications/MAMP/tmp/mysql/mysql.sock` as the socket path. If MAMP's MySQL is the only service running mysqld, this should not be necessary but YMMV...