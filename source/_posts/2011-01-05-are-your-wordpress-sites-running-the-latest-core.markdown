---
layout: post
title: "Are your WordPress sites running the latest core?"
date: 2011-01-05 20:31
comments: true
categories: 
---

In addition to hosting Drupal sites, we also host a number of WordPress sites. Similar to (checking all of our Drupal core versions)[http://echodittolabs.org/blog/2011/01/are-your-drupal-sites-running-latest-core], we needed an easy way to quickly see what versions we are running on all of our WordPress sites. Kudos to [Ethan](http://echodittolabs.org/users/ethan) for getting this one kicked off; here is a bash script to check your definable `$WEBHOME` (where you deposit all of your WordPress webroots) and scan for WordPress versions.

Note: this requires bash, grep, wc, sed, awk, and tree. If your system doesn't have tree, it's usually pretty easy to get from EPEL/Homebrew/apt-get/[mac]ports/etc.

```bash
#!/bin/bash
 
# Where to start scanning
WEBHOME=/var/www/vhosts
 
# Workaround to fix awk counting below
WEBHOMECOUNT=$(($(echo "${WEBHOME}"|grep -o "/"|wc -l| sed s/\ //g)+2))
 
# Other projects could use 'version.php' so we include 
# 'wp-includes/' in our search to limit it to WordPress
for i in $(tree -L 5 -if ${WEBHOME} | grep 'wp-includes/version.php'); do
  SITE=$(echo $i|awk -v count="$WEBHOMECOUNT" -F/ '{for(j=count;j<=NF-2;j++) \
        printf $j"/"}' | sed 's/.$//g')
  VERSION=$(grep "wp_version = " $i|awk -F\' '{print $2}')
  echo $SITE - $VERSION
done
```

As a result, you'll see output like this:

```
$ ./wordpress-version-check.sh
speedyupdates.com - 3.0.4
littlebehind.org - 3.0.3
scaredtoupgrade.com/blog - 2.8.2
kickinitoldschool.org/web/wordpress - 2.0.2
```

If you think there's a better way, or have any questions, let me know in the comments.