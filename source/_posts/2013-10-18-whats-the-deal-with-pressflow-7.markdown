---
layout: post
title: "What's the deal with Pressflow 7?"
date: 2013-10-18 09:39
comments: true
categories: 
---

Anyone who used Drupal 6 at scale knew that [Pressflow 6](http://fourkitchens.com/pressflow-makes-drupal-scale) was pretty great. It forked Drupal and added several performance enhancements, including the ability to use external caches. Since [Drupal 7 incorporated lots of the Pressflow 6 features](https://pressflow.atlassian.net/wiki/display/PF/Comparison+-+Pressflow+versus+Drupal), what's the deal with Pressflow 7?

After Drupal 7 launched, [it seemed like Pressflow 7 would continue the momentum from Pressflow 6 and greatly enhance Drupal 7](http://developmentseed.org/blog/2010/jan/07/pressflow-7-continuing-push-performance-and-scalability-drupal/). But, here we are in 2013, and you don't hear about Pressflow 7 very often, and searching for differences does not yield much more than some diffs between baseline Drupal core 7 and [the Pressflow 7 repository on GitHub](https://github.com/pressflow/7). Since there are clearly some differences, let's dive in and see what they are exactly.

- *Pressflow Smart Start* will forward you to the install page if the database is set up in settings.php but the database is empty, disabled by default. 
  - Files: [includes/bootstrap.inc](https://github.com/pressflow/7/blob/d1e3c87ab8edc207b1adbf88c6484218bcc0fa70/includes/bootstrap.inc#L2428-L2437) and [sites/default/default.settings.php](https://github.com/pressflow/7/blob/d1e3c87ab8edc207b1adbf88c6484218bcc0fa70/sites/default/default.settings.php#L555-L562)
  - Commit: https://github.com/pressflow/7/commit/fa91b2fc80741cb8c42c2db618f0ef0ad890f4cc
- Allow for environmental PRESSFLOW_SETTINGS to override settings.php; settings must be entered as a JSON array.
  - File: [includes/bootstrap.inc](https://github.com/pressflow/7/blob/d1e3c87ab8edc207b1adbf88c6484218bcc0fa70/includes/bootstrap.inc#L716-L730)
  - Commits: https://github.com/pressflow/7/commit/673fb0bdab618f8989365012149c76b8397f95d6 (Pull request: https://github.com/pressflow/7/pull/7), and https://github.com/pressflow/7/commit/953e6608b34975eaa0c8abed8a90f80d22f1b967
- APC CSS and JS check; to prevent Drupal from constantly checking the file system for core-aggregated CSS and JS files, use APC as a key:value store instead. This is very helpful improving performance for networked file systems by reducing the frequency Drupal hits the file system.
  - File: [includes/common.inc](https://github.com/pressflow/7/blob/d1e3c87ab8edc207b1adbf88c6484218bcc0fa70/includes/common.inc#L4896-L4908) (also see lines [3558](https://github.com/pressflow/7/blob/d1e3c87ab8edc207b1adbf88c6484218bcc0fa70/includes/common.inc#L3558) and [4949](https://github.com/pressflow/7/blob/d1e3c87ab8edc207b1adbf88c6484218bcc0fa70/includes/common.inc#L4949))
  - Commit: https://github.com/pressflow/7/commit/e2f9d2b10b4f0c5f61f120be145621913c6794a4 (Pull request: https://github.com/pressflow/7/pull/16/files, original pull request: https://github.com/pantheon-systems/drops-7/pull/4)
- Allow modules to act on the js_cache before writing to disk, and add a note to the aggregated JavaScript that it was built with PressFlow. This won't yield an immediate impact on anything, but adds additional functionality to modules.
  - Files: [includes/common.inc](https://github.com/pressflow/7/blob/d1e3c87ab8edc207b1adbf88c6484218bcc0fa70/includes/common.inc#L4958-L4959) and [modules/system/system.api.php](https://github.com/pressflow/7/blob/d1e3c87ab8edc207b1adbf88c6484218bcc0fa70/modules/system/system.api.php#L750-L768)
  - Commit: https://github.com/pressflow/7/commit/53098cf059236110716ca97c18565a178c458b43
- Allow sub-second delays for lock_wait(). The calculation Drupal currently does will allow a value of 0 for the lock wait time, effectively skipping it. When PHP converts to an integer, a float will always round down. For example, as Drupal performs the operation: `php -r "echo (int) 0.25 * 20;"` would return `0`. The Pressflow change corrects the order of operations and returns the correct value, for example: `php -r "echo (int) (0.25 * 20);"` would return `5`, allowing for sub-second delays to be used as an input to the function. 
  - File: [includes/lock.inc](https://github.com/pressflow/7/blob/2cd4323946987608a660694df2c02f3cb4cce6c3/includes/lock.inc#L217)
  - Commit: https://github.com/pressflow/7/commit/2cd4323946987608a660694df2c02f3cb4cce6c3

A full diff as of October 2013 can be found at https://gist.github.com/alanthing/6064500 for further reading. Original post at [drupal.stackexchange.com](http://drupal.stackexchange.com/a/80138/10865).