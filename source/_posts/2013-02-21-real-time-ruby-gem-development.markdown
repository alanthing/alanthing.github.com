---
layout: post
title: "\"Real-time\" Ruby Gem Development"
date: 2013-02-21 08:08
comments: true
categories: 
author: Alan Ivey

---

Recently, I've been working on adding functionality for [Storm on Demand](http://www.stormondemand.com)'s [API](https://www.stormondemand.com/api/docs/1.0/) to the excellent [fog](http://fog.io) library. There didn't seem to be a clear way, that I could find, to work on a Ruby gem in real-time, meaning I save the file in my editor and immediately run some code. Most [guides](http://gembundler.com/rubygems.html) I [found](http://blog.thepete.net/2010/11/creating-and-publishing-your-first-ruby.html) tended to advise to create the gem file and install it. [Stephen Ball demostrates](http://rakeroutes.com/blog/lets-write-a-gem-part-two/) the use of `bundle console` which drops you into `irb` for testing, but I was looking for a way to test my own scripts outside of a shell with my recently-saved gem files. 

Thankfully this isn't too difficult to set up and this will allow you to work on your gem in "real-time;" save and test, without `bundle console` or installing lots of dev gems.

I should note two things here: I'm not a Ruby developer, and I've really only tested this thoroughly with fog, no others. If this ends up being a terrible way to do things, please let me know.

I'm using [RVM](http://rvm.io/) to keep my gem development isolated from the rest of my system. If you haven't already, [set up rvm](https://rvm.io/rvm/install/) on your system. If you're using [rbenv](https://github.com/sstephenson/rbenv) instead, use [applicable commands](https://github.com/jamis/rbenv-gemset).

You'll obviously need to start in a gem development directory. In my case, I start with fog:

```
$ git clone git@github.com:fog/fog.git
$ cd fog
```

I'm going to switch to ruby 1.9.3 and use a new gemset called fogdev:

```
$ rvm use 1.9.3 && rvm gemset create fogdev && rvm use 1.9.3@fogdev
```

You may want to create a `.rvmrc` file so you don't have to use `rvm use` constantly.

Install the prerequesite gems as listed in the gemspec file with bundler:

```
$ bundle install
```

I'm going to grab the version number of fog from the gemspec file so I can reuse it later (if there are multiple '.gemspec' files in your directory, change '*.gemspec' to 'name.gemspec' as appropriate):

```
$ export GEMVERSION=$(grep -E '^[ \t]+s.version' *.gemspec | awk '{print $3}' | sed 's/'\''//g')
```

Now at this point, you can build and install the gem with `rake build && gem install pkg/fog-${GEMVERSION}.gem` or `gem build` or any other tools as most tutorials advise, but then whenever you edit the source files you'll have to repeat. Let's create some symbolic links instead. Start by going to the rvm gemset folder (this may be a different location for your environment):

```
$ cd ~/.rvm/gems/ruby-1.9.3-p286\@fogdev/
```

It seems as though the earlier `bundle install` command created the file bin/fog for me, but it doesn't work without either installing a gem or running the remaining commands below. Particularly for fog, if this bin/fog file is missing and you need it, you can grab it with the following:

```
$ [ ! -f bin/fog ] && curl --silent --output bin/fog https://gist.github.com/alanthing/5004411/raw/13f748f1cc19df7511ea2a01de6824eac3358905/fog && chmod +x bin/fog
```

Let's add our development gem with symbolic links (again, change as appropriate for your gem):

```
$ ln -s ~/fog/fog.gemspec specifications/fog-${GEMVERSION}.gemspec
$ ln -s ~/fog gems/fog-${GEMVERSION}
```

Now if you run `gem list` you should see your gem listed:

```
$ gem list
...snip...
ffi (1.0.11)
fission (0.4.0)
fog (1.9.0)
formatador (0.2.4)
jekyll (0.12.0)
...snip...
```

And, in my case, running `fog` uses the gemset file bin/fog and drops me into a fog-ised irb:

```
$ fog
  Welcome to fog interactive!
  :default provides VirtualBox and Vmfusion
>> 
```

If you see any mistakes or have any questions, let me know!