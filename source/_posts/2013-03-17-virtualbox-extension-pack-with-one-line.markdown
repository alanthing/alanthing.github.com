---
layout: post
title: "VirtualBox Extension Pack With One Line"
date: 2013-03-17 20:46
comments: true
categories: 
---

Just installed [VirtualBox](http://www.virtualbox.org) on OS X and need the [Extension Pack](http://www.virtualbox.org/wiki/Downloads)? This (really long) one-liner will download and install it for you. Run in terminal and enter your password once and you're off to the races!

	export version=$(/usr/bin/vboxmanage -v) && export var1=$(echo ${version} | cut -d 'r' -f 1) && export var2=$(echo ${version} | cut -d 'r' -f 2) && export file="Oracle_VM_VirtualBox_Extension_Pack-${var1}-${var2}.vbox-extpack" && curl --silent --location http://download.virtualbox.org/virtualbox/${var1}/${file} -o ~/Downloads/${file} && VBoxManage extpack install ~/Downloads/${file} --replace && rm ~/Downloads/${file} && unset version var1 var2 file

Note that VirtualBox must be installed first :)