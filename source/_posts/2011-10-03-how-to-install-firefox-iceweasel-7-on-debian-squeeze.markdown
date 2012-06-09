---
layout: post
title: "How to Install Firefox (Iceweasel) 7 on Debian Squeeze"
date: "2011-10-03 18:12:54"
comments: true
categories: Firefox Iceweasel Debian Squeze install Linux
---

First remove lagacy version of Iceweasel:

    aptitude remove iceweasel


Create `/etc/apt/sources.list.d/squeeze-backports.list` file:

    # New Mozilla packages for Squeeze
    deb http://mozilla.debian.net/ squeeze-backports iceweasel-release
    deb http://backports.debian.org/debian-backports squeeze-backports main

Then run:

    aptitude update
    aptitude install iceweasel -t squeeze-backports

Enjoy!

