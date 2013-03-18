---
layout: post
title: "Three ways to get Drush on OS X"
date: 2012-08-29 15:18
comments: true
categories:
---

[As I've said before](http://echodittolabs.org/blog/2011/03/keep-drush-up-to-date), we love [drush](http://drupal.org/project/drush). It's hard to imagine doing Drupal work without it. If OS X is your workstation, it's pretty simple to install with pear, as described on the project page. Let's review that method and cover two others, git and Homebrew, and how to keep them updated.

All commands listed below are to be executed on the command line with the Terminal application. But if you use Drush regularly then you already knew that!

## Homebrew

If you need introduced to Homebrew, or need help installing it, check out the [Homebrew project homepage](http://mxcl.github.com/homebrew/). The installation script is at the bottom.

Ready to install drush?

```
brew install drush
```

Yup, that's it! Upgrading is simple as well, as explained in the [Homebrew FAQ](https://github.com/mxcl/homebrew/wiki/FAQ):

```
brew update
brew upgrade drush
```

## PEAR

Whether you've installed your own copy of PHP with Homebrew, are using the one included with MAMP, or the version provided by OS X, drush is easy to install using the `pear` command.

If you are using the built-in php with OS X, you'll need to prefix each command with `sudo`.

```
pear channel-discover pear.drush.org
```

**Homebrew Note:** If you are using Homebrew and get an error about not being able to create a lock file, run the following command (change php54 to php53 if applicable): `touch $(brew --prefix php54)/lib/php/.lock && chmod 0644 $(brew --prefix php54)/lib/php/.lock`

Continue by installing drush now that the channel is added:

```
pear install drush/drush
pear upgrade --force Console_Getopt
```
	
**Homebrew Note:** If you do not have drush available in your path after installing, relink PHP and you should then have the symlink available in */usr/local/bin*: `brew unlink php54 && brew link php54`

Upgrading is also easy (again, if using the built-in PHP with OS X, prefix with `sudo`):

```
pear channel-update pear.drush.org
pear upgrade drush
```

## Git

If you would prefer to keep track of the source code for Drush, or perhaps be a contributor, you will want to use Git so you can easily make and keep track of changes. If you do not have git installed on your system, install [Xcode](http://itunes.apple.com/us/app/xcode/id497799835?mt=12) or [Command Line Tools for Xcode](http://developer.apple.com/downloads) and then proceed. These commands will clone the drush repository, checkout the latest stable tag, and create a symlink in */usr/local/bin*:

```
[ ! -d /usr/local ] && sudo mkdir /usr/local && sudo chgrp admin /usr/local && sudo chmod g+rwx /usr/local
git clone http://git.drupal.org/project/drush.git /usr/local/drush
cd /usr/local/drush
git checkout $(git tag|grep "^[0-9]"|egrep -v "alpha|beta|rc"|tail -1)
cd /usr/local/bin 2>/dev/null || mkdir /usr/local/bin && cd /usr/local/bin
ln -s ../drush/drush || sudo ln -s ../drush/drush
```

To update, fetch the latest code and grab the latest stable tag:

```
cd /usr/local/drush
git fetch
git checkout $(git tag|grep "^[0-9]"|egrep -v "alpha|beta|rc"|tail -1)
```

If you have any questions or improvements, please leave a comment!