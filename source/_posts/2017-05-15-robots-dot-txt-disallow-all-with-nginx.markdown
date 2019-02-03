---
layout: post
title: "Robots.txt disallow all with nginx"
date: 2017-05-15 16:16
comments: true
categories: 
---

If you're managing an environment similar to a production and want to keep bots from indexing traffic, it's customary to add a _robots.txt_ file at the root of your website to [disallow all](http://www.robotstxt.org/faq/prevent.html). Instead of creating a two-line plain text file, you can do this with only nginx:

```nginx
location = /robots.txt {
  add_header  Content-Type  text/plain;
  return 200 "User-agent: *\nDisallow: /\n";
}
```

Add this into your configuration management as determined by environment, or add it by hand, and no longer worry if Google might start broadcasting your dev site to the world.
