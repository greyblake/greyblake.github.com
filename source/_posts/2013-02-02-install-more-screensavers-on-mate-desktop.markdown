---
layout: post
title: "Install more screensavers on Mate desktop"
date: 2013-02-02 22:24
comments: true
categories: linux mate screensaver mint
---

I switched from Gnome3 to [Mate desktop](http://mate-desktop.org/)
and noticed that I'm able to use only few sreensavers.
If you're using Mate (it's default for Linux Mint) you might experience the same problem.

There is a workaround how to use much more screensavers with Mate as usually able
to do with Gnome.

Install packages with additional screensavers:

```
apt-get install xscreensaver-data-extra xscreensaver-gl-extra
```

Go to `/usr/share/applications/screensavers` directory:

```
cd /usr/share/applications/screensavers
```

There are located number of `.desktop` files. I should edit them by replacing a line

```
OnlyShowIn=GNOME;
```

with

```
OnlyShowIn=GNOME;MATE;
```.

Obviously it's a routine to change them all manually. So use `sed` tool to edit all files in once:

```sh
find . -name '*.desktop' | xargs sed -i 's/OnlyShowIn=GNOME;/OnlyShowIn=GNOME;MATE;/'
```


Now you are able to use more than hundred screensavers on Mate!
