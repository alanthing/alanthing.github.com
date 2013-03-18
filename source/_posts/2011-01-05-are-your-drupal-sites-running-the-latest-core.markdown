---
layout: post
title: "Are your Drupal sites running the latest core?"
date: 2011-01-05 19:25
comments: true
categories: 
---

At [EchoDitto](http://www.echoditto.com), we host a lot of Drupal sites. Just about all of our web servers contain more than one completely separate Drupal cores since we develop most of our websites independently. We also keep an eye out on [Drupal Security](http://drupal.org/security) and when to apply core updates. Since we're not using Aegir, nor do we run every site off of a single multi-site core install, we need a quick and easy way to see what versions we're running.

Here's a shell script we recently developed to report the Drupal Core versions for all of our sites under a single parent folder. It works on Drupal 4 through 7. It's not been tested outside of our platform, or non-RHEL systems, but it should be universal as long as you have bash, find, awk, and grep. I decided not to use [Drush](http://drupal.org/project/drush) because it would involve bootstrapping every site and invoking PHP; this is just more lightweight. Drush also would not work for Drupal 4 sites, which we surprisingly still have a couple kicking around still.

```bash
#!/bin/bash
 
# Where to start looking
WEBHOME=/var/www/vhosts
 
# Workaround to fix awk counting below, but it works
WEBHOMECOUNT=$(($(echo "${WEBHOME}"|grep -o "/"|wc -l| sed s/\ //g)+2))
 
# Change maxdepth from 5 to some other value if your webroot is not being detected
for i in $(find $WEBHOME -maxdepth 5 -type f -name system.module); do 
  # Grabs only the version number from the file as described 
  # at http://drupal.org/handbook/version-info
  VERSION=$(grep "VERSION" ${i}|awk -F\' '{print $4}')
 
  # Drupal 4 has the system.module file in a lower directory than 5+
  if [[ $VERSION == 4* ]]; then
    SITE=$(echo $i|awk -v count="$WEBHOMECOUNT" -F/ '{for(j=count;j<=NF-2;j++) \
           printf $j"/"}' | sed 's/.$//g')
  else
    SITE=$(echo $i|awk -v count="$WEBHOMECOUNT" -F/ '{for(j=count;j<=NF-3;j++) \
           printf $j"/"}' | sed 's/.$//g')
    if [ -z $VERSION ]; then
      # Drupal 7 does not keep the version number on system.module, despite the 
      # drupal.org page. Grab from changelog (disabled b/c it includes extra info)
      #VERSION="$(sed '3q;d' ${WEBHOME}/${SITE}/CHANGELOG.txt)"
      # All versions of Drupal have the version of the module in system.info, 
      # so we grab it here for D7. The only reason I'm not using this for all 
      # versions is because of the aforementioned drupal.org page
      VERSION=$(grep "version = \"" ${WEBHOME}/${SITE}/modules/system/system.info \
                | awk -F\" '{print $2}')
    fi
  fi
  echo $SITE - $VERSION
done
```

As a result, you'll see the directory tree after `$WEBHOME` where the root of the site is and the version number. Below is an example:

```
$ ./drupal-version-check.sh
mysite.org/www - 5.22
thissite.net - 4.7.11
subfolder/drupalftw.com - 6.12
blah.org/trunk - 7.0
```

If you have suggestions, please send them over. This script is still young and I'd love to see if anyone has any better ideas.