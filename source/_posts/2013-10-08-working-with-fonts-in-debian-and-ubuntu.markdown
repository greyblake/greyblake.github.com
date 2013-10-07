---
layout: post
title: "Working with fonts in Debian and Ubuntu"
date: 2013-10-08 00:43
comments: true
categories: linux debian ubuntu fonts
---

## Install fonts

There are a lot of fonts in standard Debian repository. Packages which contains
fonts starts with `fonts-`, so lets install them all. Run the next command as
root:

```bash
apt-cache search ^fonts- | sed 's/^\(fonts-[^ ]*\).*$/\1/' | xargs apt-get install
```

Short explanation of the command:

* `apt-cache search ^fonts-` - find all packages which starts with `fonts-`;
* `sed 's/^\(fonts-[^ ]*\).*$/\1/'` - filter output to get only package names;
* `xargs apt-get install` - pass package names to `apt-get install` to install them.

## Preview fonts

Now you have more than 1500 fonts, but it's hard to pick one that you need, because
it's hard to look through all of them. For our luck there exist specials to preview
fonts, and one of is called `fontmatrix`. Lets install it:

```bash
apt-get install fontmatrix
```

And run it:

```
fontmatrix
```

![Fontmatrix](http://i1078.photobucket.com/albums/w484/greyblake/fontmatrix.png)

Now it's much easier to select right font!:)
