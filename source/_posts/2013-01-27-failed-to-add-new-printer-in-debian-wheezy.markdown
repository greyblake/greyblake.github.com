---
layout: post
title: "Failed to add new printer in Debian Wheezy"
date: 2013-01-27 21:44
comments: true
categories: linux debian wheezy printer
---


After migrating to Debian Wheezy (current test Debian repository) I faced a problem
with installing a local USB printer: on attempt to add new USB printer
(HP LaserJet M1005) I got an error message: "Failed to add new printer in Debian Wheezy"
("Не удалось добавить новый принтер" if you have Russian localization).

After googling for I a while I found a good workaround for it. All you need is
to use `pk-helper` package from Sid (unstable Debian version), since it seems to
be buggy in Wheezy.

So add Sid repository to your `/etc/apt/source.list`:

```
deb http://ftp.ua.debian.org/debian/ sid main non-free contrib
```

And reinstall `cups-pk-helper` package using Sid repository:


```
aptitude reinstall cups-pk-helper --target sid
```

That's all. Now try to connect your USB printer.
Btw, there is the [bug report](http://bugs.debian.org/cgi-bin/bugreport.cgi?bug=695131).
