---
layout: post
title: "OS X 10.7 Lion Development: Nginx, PHP, MariaDB with Homebrew"
date: 2012-04-22 20:41
comments: true
categories:
---

*This post originally featured on the [Echo &amp; Co. blog](http://echo.co/blog/os-x-107-lion-development-nginx-php-mariadb-homebrew)*.

[Nginx](http://wiki.nginx.org) is quickly becoming a popular, low resource alternative to Apache for many websites. This doesn't come without challenges, such as using PHP as CGI due to not having mod_php available. Nginx also does not use any Apache configuration rules, nor does it use .htaccess or anything like it, so it requires additional configuration regardless of the web application being deployed. A big help in getting Nginx started with Drupal is [António P. P. Almeida's](https://github.com/perusio) [drupal-with-nginx configuration](https://github.com/perusio/drupal-with-nginx/), which makes it fairly simple to deploy in Linux. But what about local development on OS X? Read on to learn get all of the required components set up for your system, as well as the modifications necessary to get drupal-with-nginx set up on OS X.

## Prerequisites

We'll be building all of our packages with [Homebrew](http://mxcl.github.com/homebrew/), which is, in my opinion, one of the best ways to easily add lot's of great open-source software on OS X. Homebrew requires that you have a compiler, so you can either install the huge [Xcode package](http://itunes.apple.com/us/app/xcode/id448457090), or I would recommend Apple's [Xcode Command Line Tools](http://kennethreitz.com/xcode-gcc-and-homebrew.html) which is a much smaller download and officially supported by Homebrew.

Once you have either Xcode or Xcode Command Line Tools installed, [install Homebrew](https://github.com/mxcl/homebrew/wiki/installation).

Note that for all commands below that are starting with a **$**, the dollar sign is showing a command-line prompt in [Terminal](http://www.apple.com/macosx/apps/all.html#terminal), and you should not actually type it as part of the commands. I also make heavy use of `$(brew --prefix)` to make these instructions persist passed current Homebrew formula versions, and hopefully also for an installation with Homebrew in a path other than /usr/local, though I have not tested it.

Also note that many times in this post you will see **/n**; make sure you type those or include them with your copy and paste, they are not CMS errors :)

## Database: MariaDB

I've [covered before why I like MariaDB](http://echodittolabs.org/blog/2011/09/os-x-107-lion-development-native-apache-php-homebrew-mysql-or-mariadb), but you could easily swap this out with MySQL if you would rather. We'll start by installing MariaDB with Homebrew.

```bash
brew install mariadb pidof (note: OS X may ask you if you want to install 'javac')
unset TMPDIR 
mysql_install_db --verbose --user=`whoami` --basedir="$(brew --prefix mariadb)" --datadir=$(brew --prefix)/var/mysql --tmpdir=/tmp 
cp $(brew --prefix mariadb)/share/mysql/my-small.cnf $(brew --prefix mariadb)/my.cnf 
sed -i "" 's/max_allowed_packet = 1.*M/max_allowed_packet = 2G/g' $(brew --prefix mariadb)/my.cnf 
[ ! -d ~/Library/LaunchAgents ] && mkdir ~/Library/LaunchAgents 
[ -f $(brew --prefix mariadb)/homebrew.mxcl.mariadb.plist ] && cp -v $(brew --prefix mariadb)/homebrew.mxcl.mariadb.plist ~/Library/LaunchAgents/ 
[ -f ~/Library/LaunchAgents/homebrew.mxcl.mariadb.plist ] && launchctl load -w ~/Library/LaunchAgents/homebrew.mxcl.mariadb.plist 
$(brew --prefix mariadb)/bin/mysql_secure_installation 
```
Note: you could alternatively run: `$(brew --prefix mariadb)/bin/mysqladmin -u root password 'new-password'` instead of `mysql_secure_installation`

## PHP

OS X comes with PHP installed, but it doesn't come with PHP-FPM. While it's likely possible to run PHP as FastCGI with the built-in OS X, I prefer to install PHP with Homebrew since we're using Homebrew for everything else, and it keeps everything self-contained in Homebrew's root (defaults to /usr/local). Note that the `brew tap` command requires [Homebrew 0.9 or greater](https://github.com/mxcl/homebrew/wiki/Homebrew-0.9).

```bash
brew tap josegonzalez/php 
brew install php --with-mariadb --with-suhosin --with-fpm 
mkdir -v $(brew --prefix php)/var/log 
cp -v $(brew --prefix)/etc/php-fpm.conf.default $(brew --prefix)/etc/php-fpm.conf 
sed -i '' 's|;\(daemonize[[:space:]]*=[[:space:]]*\)yes|\1no|g' $(brew --prefix)/etc/php-fpm.conf 
[ ! -d ~/Library/LaunchAgents ] && mkdir ~/Library/LaunchAgents 
[ -f $(brew --prefix php)/org.php-fpm.plist ] && cp -v $(brew --prefix php)/org.php-fpm.plist ~/Library/LaunchAgents/ 
[ -f ~/Library/LaunchAgents/org.php-fpm.plist ] && launchctl load -w ~/Library/LaunchAgents/org.php-fpm.plist
```

I would recommend the following settings for full compatibility with drupal-with-nginx, and to set the time zone to silence a lot of PHP warnings:

```bash
sed -i '' 's|;\(pm.status_path[[:space:]]*=[[:space:]]*/\)\(status\)|\1fpm-\2|g' $(brew --prefix)/etc/php-fpm.conf 
sed -i '' 's|;\(ping.path[[:space:]]*=[[:space:]]*/ping\)|\1|g' $(brew --prefix)/etc/php-fpm.conf 
sed -i '' 's|;\(ping.response[[:space:]]*=[[:space:]]*pong\)|\1|g' $(brew --prefix)/etc/php-fpm.conf 
sed -i '' "s|;\(date\.timezone[[:space:]]*=\).*|\1 $(php -d 'error_reporting=' -r 'echo date("e", time());')|g" $(brew --prefix)/etc/php.ini
```

By default, PHP-FPM runs on a socket, which means that connections to PHP-FPM will require using TCP. You also have the option to use Unix sockets, which means slightly less overhead in PHP-FPM connections. Note that the drupal-with-nginx repository is set up for TCP by default, though if you choose to run the following command I will tell you how to use Unix sockets with Nginx.

```bash
sed -i '' 's|\(listen[[:space:]]*=[[:space:]]*\)127.0.0.1:9000|\1var/www.sock|g' $(brew --prefix)/etc/php-fpm.conf
```

### Optional: PHP Extensions

The [third-party Homebrew keg that we "tapped" into](https://github.com/josegonzalez/homebrew-php) also provides easy formulas for PHP extensions. None of these are required to run Nginx and Drupal locally. You may also note that *uploadprogress* does not work with anything but mod_php, but by installing it now you could theoretically use the same PHP installation with Apache if you wanted and already have it ready to go. Feel free to omit it, or any of these below, though I would at least recommend APC for performance reasons.

```bash
brew install uploadprogress-php 
echo -e "\n[uploadprogress]\nextension=\"$(brew --prefix uploadprogress-php)/uploadprogress.so\"" >> $(brew --prefix)/etc/php.ini 
brew install apc-php 
echo -e "\n[apc]\nextension=\"$(brew --prefix apc-php)/apc.so\"\napc.enabled=1 \napc.shm_segments=1 \napc.shm_size=64M \napc.ttl=7200 \napc.user_ttl=7200 \napc.num_files_hint=1024 \napc.mmap_file_mask=/tmp/apc.XXXXXX \napc.enable_cli=1" >> $(brew --prefix)/php.ini 
brew install memcache-php 
echo -e "\n[memcache]\nextension=\"$(brew --prefix memcache-php)/memcache.so\"" >> $(brew --prefix)/etc/php.ini 
echo -e "memcache.hash_strategy=\"consistent\"" >> $(brew --prefix)/etc/php.ini 
brew install xdebug-php 
echo -e "\n[xdebug]\nzend_extension=\"$(brew --prefix xdebug-php)/xdebug.so\"" >> $(brew --prefix)/etc/php.ini 
brew install xhprof-php 
echo -e "\n[xhprof]\nextension=\"$(brew --prefix xhprof-php)/xhprof.so\"" >> $(brew --prefix)/etc/php.ini 
```

Once you've finished configuring PHP, you can reload the settings for PHP-FPM (or, you could find the pid of the first php-fpm process and send it the SIGUSR2 signal; this is easier):

```bash
[ -f ~/Library/LaunchAgents/org.php-fpm.plist ] && launchctl unload -w ~/Library/LaunchAgents/org.php-fpm.plist && launchctl load -w ~/Library/LaunchAgents/org.php-fpm.plist
```

## Nginx

After compiling MariaDB and PHP, you're probably not too excited about compiling another application. Luckily, Nginx is a faily quick build, at least compared to MariaDB and PHP. We'll include some build options not on by default since they add features references in drupal-with-nginx, and we'll also add some 3rd party extensions as well. We'll start by grabbing those extensions:

```bash
curl -s -L -o /tmp/nginx-upload-progress.tar.gz https://github.com/masterzen/nginx-upload-progress-module/tarball/v0.9.0 && mkdir /tmp/nginx-upload-progress && tar zxpf /tmp/nginx-upload-progress.tar.gz --strip-components 1 -C /tmp/nginx-upload-progress && rm /tmp/nginx-upload-progress.tar.gz 
curl -s -L -o /tmp/nginx-fair.tar.gz http://github.com/gnosek/nginx-upstream-fair/tarball/master && mkdir /tmp/nginx-fair && tar zxpf /tmp/nginx-fair.tar.gz --strip-components 1 -C /tmp/nginx-fair && rm /tmp/nginx-fair.tar.gz
```

The next section is one giant line of sed regex that will edit the Homebrew formula for nginx to add the additional compile options that we need. Make sure it all gets entered as one line (yes, you should use copy and paste here)!

```bash
sed -i '-default' 's/\([[:space:]]*\['\''--\)\(with-webdav\)\('\'',[[:space:]]*"\)\(Compile with support for WebDAV module\)\("\]\)/\1\2\3\4\5,%\1with-realip\3Compile with support for RealIP module\5,%\1with-gzip_static\3Compile with support for Gzip Static module\5,%\1with-uploadprogress\3Compile with support for Upload Progress module\5,%\1with-fair\3Compile with support for Fair module\5,%\1with-mp4\3Compile with support for MP4 module\5,%\1with-flv\3Compile with support for FLV module\5,%\1with-stub_status\3Compile with support for Stub Status module\5/; s/\([[:space:]]* args << "--\)\(with-http_dav_module\)\(" if ARGV.include? '\''--with-\)\(webdav\)\('\''.*\)/\1\2\3\4\5%\1with-http_realip_module\3realip\5%\1with-http_gzip_static_module\3gzip_static\5%\1add-module=\/tmp\/nginx-upload-progress\3uploadprogress\5%\1add-module=\/tmp\/nginx-fair\3fair\5%\1with-http_mp4_module\3mp4\5%\1with-http_flv_module\3flv\5%\1with-http_stub_status_module\3stub_status\5/; y/%/\n/' $(brew --prefix)/Library/Formula/nginx.rb
```

Now we'll install Nginx with our new build options and extensions and start it.

```bash
brew install nginx --with-realip --with-gzip_static --with-mp4 --with-flv --with-stub_status --with-uploadprogress --with-fair 
[ $? -eq 0 ] && rm -rf /tmp/nginx-upload-progress /tmp/nginx-fair 
mkdir -vp $(brew --prefix nginx)/var/{microcache,log,run} 
[ ! -d ~/Library/LaunchAgents ] && mkdir ~/Library/LaunchAgents 
[ -f $(brew --prefix nginx)/homebrew.mxcl.nginx.plist ] && cp -v $(brew --prefix nginx)/homebrew.mxcl.nginx.plist ~/Library/LaunchAgents/ 
[ -f ~/Library/LaunchAgents/homebrew.mxcl.nginx.plist ] && launchctl load -w ~/Library/LaunchAgents/homebrew.mxcl.nginx.plist
```

## Nginx-With-Drupal

We're now ready to set up Nginx to work with Drupal. [I've created a fork of the original repository](https://github.com/alanthing/drupal-with-nginx/tree/osx-1.0.x) to make some necessary changes for OS X. I also make several changes for the stable 1.0.x branch of Nginx, which we've just installed, rather than the unstable 1.1.x branch that is in the original configuration.

If you want to stick with the original project, [check out the changes I made for OS X and 1.0.x](https://github.com/alanthing/drupal-with-nginx/commit/f13f7d8ce9af0853a5601dcbb90c5e0c2ed8dadd) and you can apply them yourself and skip the git steps below.

```bash
[ -d $(brew --prefix)/etc/nginx ] && mv -v $(brew --prefix)/etc/nginx $(brew --prefix)/etc/nginx-default 
git clone https://github.com/alanthing/drupal-with-nginx.git $(brew --prefix)/etc/nginx 
cd $(brew --prefix)/etc/nginx 
git checkout osx-1.0.x 
mkdir sites-enabled 
cd sites-enabled 
ln -s ../sites-available/000-default 
cp -a ../sites-available/example.com.conf yournewsite.conf
```

That's about as much as I can automate for you with copy+paste-able commands! You'll want to do the following to *yournewsite*.conf, which you can rename to be anything, to configure Nginx for your website:

*	Change `server_name`, `access_log`, and `error_log` to use your local domain name for your virtual host
*	Change root to the path of your Drupal installation. On my system, this may be */Users/alan/Sites/drupal-7.14*. Note that you cannot use *~* in place of */Users/name*
*	Unless you have a valid SSL certificate, you'll probably want to completely delete the second half of the file. So find the line containing `} HTTP server` and remove all following lines
*	If you're using Boost with Drupal 7, or Drupal 6 with/without Boost, note that you'll want to comment out the `include sites-available/drupal.conf` line and uncomment the other relevant one for your site
*	If you want to get additional status messages from PHP-FPM, uncomment `include php_fpm_status_vhost.conf` in this file, and also `include php_fpm_status_allowed_hosts.conf` in `$(brew --prefix)/etc/nginx/nginx.conf`
*	If you configured PHP-FPM earlier to use Unix sockets instead of TCP, open nginx.conf and comment out `include upstream_phpcgi_tcp.conf` and uncomment `include upstream_phpcgi_unix.conf`
*	Depending on the location of your files directory, you may need to edit the `sites-available/drupal.conf` (or other drupal*.conf) file and change the relevant sites/default/files paths appropriately

Once you're finished editing your virtual host conf file (and possibly nginx.conf), you can reload nginx easily:

```bash
$(brew --prefix)/sbin/nginx -s reload
```

## Bonus: Drush

You'll need Drush to install a new Drupal site, as install.php is blocked for security reasons by default. Also, you'll find cron.php is inaccessible as well. There's a weird little hack required to get Drush installed with Homebrew without requiring sudo, so below is an example of how to both get around the sudo requirement and set up a blank Drupal 7 website (note that using the root MySQL user is bad form, but this is meant more as a quick demo of using drush with this setup than a recommended setup).

```bash
touch $(brew --prefix php)/lib/php/.lock 
chmod 0644 $(brew --prefix php)/lib/php/.lock 
$(brew --prefix)/bin/pear channel-discover pear.drush.org 
$(brew --prefix)/bin/pear install drush/drush 
brew unlink php && brew link php 
cd ~/Sites 2>/dev/null || mkdir ~/Sites && cd ~/Sites 
drush dl 
mysql -uroot -p'yourpassword' -e'create database drupal;' 
cd drupal-7.14 
drush si --db-url=mysql://root:yourpassword@localhost/drupal
```