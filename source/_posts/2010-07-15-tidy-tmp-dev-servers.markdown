---
layout: post
title: "Tidy up /tmp on dev servers"
date: 2010-07-15 16:53
comments: true
categories:
---

*This post originally featured on the [Echo &amp; Co. blog](http://echo.co/blog/tidy-tmp-dev-servers)*.

Today I was made aware of some strange MySQL errors on our development server that ended up being related to a lack of sufficient disk space. Upon inspection, the /tmp partition had completely filled up. A majority of the files were temporary files created by the [Drupal module devel](http://drupal.org/project/devel) with devel_themer enabled. Turns out [these temporary files are not deleted at the end of a session](http://drupal.org/node/327512). There were also CURLCOOKIE files related to cron jobs to hit cron.php on various dev and production sites, and some other temporary files that if left unattended could result in the same errors.

My solution is to have these files deleted every night, but only if they're three days old, which should be adequate time for the developer to need access to these temporary files. I added this to root's cron.

```
# Delete some /tmp files older than 3 days
5 4 * * * find /tmp -type f -name "Apache-Session*" -mtime +3 -exec rm -f {} \;
6 4 * * * find /tmp -type f -name "backup_migrate_*" -mtime +3 -exec rm -f {} \;
7 4 * * * find /tmp -type f -name "devel_themer_*" -mtime +3 -exec rm -f {} \;
8 4 * * * find /tmp -type f -name "CURLCOOKIE_*" -mtime +3 -exec rm -f {} \;
```

Similarly, I have a cron job to remove old Apache logs, which is helpful since an old project will never have it's logs deleted unless they're removed manually.

```
# Delete Apache logs older than 30 days
30 3 * * * find /var/log/httpd -type f -mtime +30 -exec rm -f {} \;
```

Do you have other ideas for managing temporary files that don't clean up after themselves? Let me know in the comments.