---
layout: post
title: "Use Syslog instead of Watchdog"
date: 2011-08-17 16:02
comments: true
categories:
---

In this example, we choose one of our servers to receive the Syslog traffic and the others will send traffic to it. In a large environment, you should use a non-public-facing server to act as a syslog server for your sites, ideally using a database to store data, but we'll just output everything to a file to keep things simple for this guide. If you decide to use a database like MySQL to store your data, this can still provide you with a starting point, but just know that using your same production database server defeats my intended purpose since I was ultimately trying to alleviate the amount of writes on the database. 

Start by using yum to install **rsyslog**. If you're using another distro, rsyslog is likely included in the base repositories. Also, note that CentOS 6 uses rsyslog by default so you may skip this step if it's already installed on your system. You'll need this on all servers running Drupal or your rsyslog server:

```bash
yum install rsyslog
```

For CentOS 5, you'll need to unload the default syslog and switch to rsyslog. The following will ensure it starts on boot as well. Until otherwise noted, run the following commands on all servers:

```bash
/sbin/service syslog stop && /sbin/service rsyslog start
/sbin/chkconfig rsyslog on
/sbin/chkconfig syslog off
```

Rather than hacking the config file to bits, I find it easier to create an include folder and keep things organized by files. I also like to back up default files created by RPMs so if an update leaves behind rsyslog.rpmnew, you can compare it to the default to see the changes:

```bash
mkdir -v /etc/rsyslog.d
cp -av /etc/rsyslog.conf{,-default}
```

Add the following to **/etc/rsyslog.conf** below the last line containing "$ModLoad":

```
# Include all files in /etc/rsyslog.d
$IncludeConfig /etc/rsyslog.d/*.conf
```

Reload the current rsyslog configuration:

```bash
/sbin/service rsyslog reload
```

The next several steps will only need to be run on the syslog server. Back up and edit **/etc/sysconfig/rsyslog** to accept UDP connections from other servers. If you're using iptables, open up port 514 on UDP.

```bash
cp -a /etc/sysconfig/rsyslog /etc/sysconfig/rsyslog-default
```

Edit /etc/sysconfig.rsyslog and change **SYSLOGD_OPTIONS="-m 0"** to **SYSLOGD_OPTIONS="-m 0 -r514"**

Create a root folder for website logs to go:

```bash
mkdir /var/log/websites
```

Create the following file to work with example.com (this format will allow you to create other files for anotherexample.org, etc.): **/etc/rsyslog.d/example.conf**. Add the following:

```
# example.com
$template DailyPerHostLogs-example,"/var/log/websites/%SYSLOGTAG:F,58:1%/%SYSLOGTAG:F,58:1%.%$YEAR%-%$MONTH%-%$DAY%.log"
:syslogtag, contains, "example"                    -?DailyPerHostLogs-example
& ~
```

This will create **/var/log/websites/example/example.2011-08-17.log** and new files for each day. To add other websites, duplicate the file and replace example with another string. We'll later use "example" in the Drupal configuration for Syslog later that will correlate to the correct file.

For those curious about this configuration: a template is needed for dynamic file paths with year-month-day. The syslogtag variable will use the string sent by Drupal in the Syslog module settings. Drupal will send it as **example:**, so *:F,58:1* will use the colon as the field separator and grab the first field. Even though it would appear as though this single template could be used on multiple sites, I found I still had to have a unique template for each website or all websites would dump into the same file (perhaps this is remedied in newer versions of rsyslog). The last line means that the syslog input will not be sent to any other syslog files.

Reload the current rsyslog configuration, and the server is set up:

```bash
/sbin/service rsyslog reload
```

On the clients, we'll create simple .conf files for each website. Create **/etc/rsyslog.d/example.conf** on each additional web server containing:

```
# example.com
:syslogtag, contains, "example"     @192.168.1.20
```

Note that the IP address can also be a domain name, though I would not recommend it to keep another process from being needed for the lookup.

Reload the current rsyslog configuration, and each web client is set up:

```bash
/sbin/service rsyslog reload
```

Enable the syslog module and go to **http://example.com/admin/settings/logging/syslog**. Set the Syslog Identity to match the value used in /etc/rsyslog.d/example.conf; in this case, **example**. Using the rsyslog filter we're using, it does not matter which facility we send to. Save the configuration.

![rsyslog-screenshot](http://echoditto.com/sites/default/files/rsyslog-screenshot.png)

Go to your rsyslog server and watch the **/var/log/websites/example/example.2011-08-16.log** file. After verifying that the server is receiving all messages, disable the Watchdog module in **http://example.org/admin/build/modules/list**.</p>

There are improvements that can be had here, like using TCP instead of UDP, using [LogAnalyzer](http://loganalyzer.adiscon.com) to filter through the files, and more. This was meant to be an introduction into getting combined text files of your watchdog output while reducing the strain on your database by not writing to it as often.