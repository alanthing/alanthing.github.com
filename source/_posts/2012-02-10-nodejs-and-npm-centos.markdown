---
layout: post
title: "Node.js and NPM on CentOS"
date: 2012-02-10 17:23
comments: true
categories:
---

***Update:* This no longer appears to be necessary as of nodejs 0.8.0.** It may have been fixed earlier but I noticed neither of these changes are necessary anymore. Something new though, I had problems with node-gyp, and the solution was to install python26 with yum and then re-run the npm command with `PYTHON="/usr/bin/python26" npm install -deps` or similar.

The preferred way to install node and NPM seems to be installing from source, but I'm a perennial fan of using packages to keep things tidy, especially if I need to uninstall something. I started by going to the [Node.js download page](http://nodejs.org/#download), and through to [Installing with a package manager](https://github.com/joyent/node/wiki/Installing-Node.js-via-package-manager). I installed the yum release RPM for the [tchol.org](http://nodejs.tchol.org/) repository as directed and installed **nodejs** and **npm** with yum. From there, I ran into two problems but thankfully they were fairly easy to resolve.

The first thing I wanted to do was install [forever](https://github.com/nodejitsu/forever) globally so I could use it to keep applications running persistently. But running `npm install forever -g` kept stalling. The npm RPM installed from the tchol.org repository creates the symlink /usr/lib/node_modules to /usr/lib/nodejs. That's fine, but /usr/lib/nodejs is owned by root:root. Running a npm install command with sudo attempted to set the ownership of the NPM modules as nobody:user, and NPM wasn't exiting due to permissions for some reason.

I have ACLs enabled on my file system, so I fixed this by allowing nobody write access to /usr/lib/nodejs:

	sudo setfacl -m u:nobody:rwx /usr/lib/nodejs
	
If you don't have ACLs enabled on your filesystem, you could allow nobody to be the folder owner:

	sudo chown nobody:nobody /usr/lib/nodejs

The second problem I ran into was, after installing forever, it wouldn't run. The nodejs package installed the binary as /usr/bin/nodejs but /usr/bin/forever begins with **#!/usr/bin/env node**, which will not return a valid interpreter. I wanted to install a symlink into /usr/local/bin, but some users don't always have that in their path, so I created a link via:

	sudo ln -s /usr/bin/nodejs /usr/bin/node

Now I'm able to install anything I need to with NPM and run my Node.js applications with Forever. I'm still very new to Node, so if you think I should be doing things a better way, let me know in the comments!