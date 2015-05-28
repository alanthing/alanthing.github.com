---
layout: post
title: "Never touch your local /etc/hosts file in OS X again"
date: 2012-04-24 08:20
comments: true
categories:
---

In each of my posts on setting up a [local](http://echodittolabs.org/blog/2012/04/os-x-107-lion-development-nginx-php-mariadb-homebrew) [development](http://echodittolabs.org/blog/2011/10/os-x-107-lion-development-macports) [environment](http://echodittolabs.org/blog/2011/09/os-x-107-lion-development-native-apache-php-homebrew-mysql-or-mariadb) [on OS X](http://echodittolabs.org/blog/2011/08/os-x-107-lion-development-native-mamp-mysql-installer), it's mentioned that you need to add your website's domain, even though it's local, in your /etc/hosts file. My preferred way to edit the hosts file on OS X is using [Gas Mask](http://code.google.com/p/gmask/). If you wanted to create the local virtual host **projectx.dev**, you would add the line `127.0.0.1 projectx.dev` in /etc/hosts or with Gas Mask, and then use that same value in either **ServerName** in Apache or **server_name** in Nginx. This can be tedious for adding new sites. Luckily there's a way to set this up once and then never have to edit your hosts file again for adding new local virtual hosts.

You'll need a copy of dnsmasq, and I find this is most easily installed via Homebrew. If you haven't already, grab either [Xcode](http://itunes.apple.com/us/app/xcode/id448457090) or [Xcode Command Line Tools](http://kennethreitz.com/xcode-gcc-and-homebrew.html) and [install Homebrew](https://github.com/mxcl/homebrew/wiki/installation).

The steps below will install dnsmasq from Homebrew, configure dnsmasq to return the IP address '127.0.0.1' for all requests to the fake top-level-domain ".dev," start dnsmasq on boot (don't worry, it's an extremely light-weight process), and configure OS X to use dnsmasq for queries ending in ".dev."

```
brew install dnsmasq
mkdir -pv $(brew --prefix)/etc/
echo 'address=/.dev/127.0.0.1' > $(brew --prefix)/etc/dnsmasq.conf
sudo cp -v $(brew --prefix dnsmasq)/homebrew.mxcl.dnsmasq.plist /Library/LaunchDaemons
sudo launchctl load -w /Library/LaunchDaemons/homebrew.mxcl.dnsmasq.plist
sudo mkdir -v /etc/resolver
sudo bash -c 'echo "nameserver 127.0.0.1" > /etc/resolver/dev'
```

That's it! You can run `scutil --dns` to show all of your current resolvers, and you should see that all requests for a domain ending in .dev will go to the DNS server at 127.0.0.1:

```
resolver #9
  domain   : dev
  nameserver[0] : 127.0.0.1
```

If you ping any domain that ends in .dev, you'll get the IP address 127.0.0.1 back as a result:

```
$ ping -c 1 thereisnowaythisisarealdomain.dev
PING thereisnowaythisisarealdomain.dev (127.0.0.1): 56 data bytes
64 bytes from 127.0.0.1: icmp_seq=0 ttl=64 time=0.057 ms
```

Note that if you're using Mountain Lion you may need to reboot before the /etc/resolver settings take effect globally. I was able to get pings to work right away but Chrome would not resolve properly until I restarted.

Now you can set up a new virtual host with **anydomain.dev**, and it'll be available as soon as you reload Apache or Nginx! You could extend this by enabling [Mass Virtual Hosting for Apache](http://httpd.apache.org/docs/2.2/vhosts/mass.html) or [similar with Nginx](http://forum.nginx.org/read.php?2,218617,218617#msg-218617), though both require consistent layout of directories. Have fun configuring less stuff on your system!