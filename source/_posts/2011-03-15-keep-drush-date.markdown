---
layout: post
title: "Keep Drush Up to Date"
date: 2011-03-25 17:39
comments: true
categories:
---

**UPDATE:** See [Three Ways to Get Drush on OS X](/blog/2012/08/29/three-ways-get-drush-os-x/) for a newer guide.

At [EchoDitto](http://www.echoditto.com), we're big fans of [Drush](http://drupal.org/project/drush). It's installed on all of our servers and it's a great way to [perform maintenance tasks](http://drush.ws/#core-cron), [download core and modules](http://drush.ws/#pm-download), and [much much more](http://www.lullabot.com/articles/drush-make-and-pressflow). I'm not a big fan of installing from zip files though, so let's use git to easily keep our Drush install up to date.

Before [the git transition](http://www.lullabot.com/blog/git-coming-soon-drupalorg), I would use Subversion to check out the latest drush release somewhere, like /usr/local/drush (on my Mac), using the CVS clone [Subversible](http://dembach.net/subversible/en). Today, however, I noticed that the latest tagged release Subversible has is [4.2](https://subversible.svn.beanstalkapp.com/modules/drush/tags/), and the [Drush project page](http://drupal.org/project/drush) has 4.4 ready to download. I figured now would be a good time to start using [git](http://git-scm.com/) for this and stop relying on a third-party to provide the code for Subversion.

The commands below will walk through cloning the Drush git repository and then switching to the latest release. I realize the [drush self-update](http://drush.ws/#self-update) will soon make this irrelevant, but this method works today and is a great exercise in learning how Drupal is using git, especially important coming from Subversion.

Clone the repository. It will create the subfolder "drush" from your current working directory:

```bash
git clone --branch master http://git.drupal.org/project/drush.git
cd drush
```

Use pear to download *includes/table.inc*. This one-liner will take care of everything for you as long as you have pear installed and are in the new drush directory:

```bash
pear download Console_Table
tar zxvf `ls Console_Table-*.tgz` `ls Console_Table-*.tgz | sed -e 's/\.[a-z]\{3\}$//'`/Table.php
mv `ls Console_Table-*.tgz | sed -e 's/\.[a-z]\{3\}$//'`/Table.php includes/table.inc
chmod 0644 includes/table.inc && rm -fr Console_Table-*
```

At this point you may or may not have a functioning Drush installation, because you're using the HEAD of the Drush code repository and it might contain broken code. I would recommend switching the checkout to the latest release. We'll start by viewing all releases:

```
$ git tag
5.x-1.0
5.x-1.0-beta1
5.x-1.0-beta2
... snip ...
7.x-4.2
7.x-4.3
7.x-4.4
debian/4.3-1
debian/4.4-1
```

As of this writing, there are over 50 results from this command, but you'll see the latest is "7.x-4.4". If we want our local Drush install to be at 4.4, change the checkout:

```
$ git checkout 7.x-4.4
Previous HEAD position was 1532d0c... #1079434 by msonnabaum. Add .gitignore for includes/table.inc
HEAD is now at a808ff0... Updating drush version
```

That's it! Drush is set up on 4.4:

```
$ drush --version
drush version 4.4
```

Let's say 4.5 is released tomorrow and you want to change your checkout. First you'll need to update the local git repository: `git pull`

View the list of tags:

```
$ git tag
5.x-1.0
5.x-1.0-beta1
5.x-1.0-beta2
... snip ...
7.x-4.3
7.x-4.4
7.x-4.5
debian/4.3-1
debian/4.4-1
debian/4.5-1
```

And switch your checkout to the new release: `git checkout 7.x-4.5`

Repeat for future releases, and enjoy! Of course, once [drush self-update](http://drush.ws/#self-update) begins working, that will be the preferred method, but this will work today. Please questions or comments below.