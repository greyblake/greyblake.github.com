---
layout: post
title: "How to build Vim against specific Ruby version"
date: 2012-07-15 23:00
comments: true
categories: Ruby Vim make ruby1.9
---

Let's say you installed your Vim editor as a system package. Likely it was compiled against Ruby 1.8.7.
But what if you need Vim compiled with Ruby 1.9.x?
I'm gonna tell you how to do it.

<!--more-->

## Clone vim repository

Make sure you have installed `mercurial` package:

```sh
$ sudo apt-get install mercurial
```

Clone Vim repository from google code:

```sh
$ hg clone https://vim.googlecode.com/hg/ vim_sources
$ cd vim_sources
```

## Patching

There is no option to specify Ruby version to compile Vim against.
But it's easy to patch vim sources to force it to use `1.9` instead of `1.8`.

Open file `src/Make_mvc.mak` in vim sources directory.

Find lines like this:


```make src/Make_mvc.mak

#
# Support Ruby interface
#
!ifdef RUBY
#  Set default value
!ifndef RUBY_VER
RUBY_VER = 18
!endif
!ifndef RUBY_VER_LONG
RUBY_VER_LONG = 1.8
!endif

```


Replace `18` and `1.8` with `19` and `1.9`. So it should look this way:

```make src/Make_mvc.mak

#
# Support Ruby interface
#
!ifdef RUBY
#  Set default value
!ifndef RUBY_VER
RUBY_VER = 19
!endif
!ifndef RUBY_VER_LONG
RUBY_VER_LONG = 1.9
!endif

```

## Configure and compile

At first make sure your current ruby version is which one you need:

```sh
$ rvm use 1.9.3 --default
$ ruby -v
ruby 1.9.3p194 (2012-04-20 revision 35410) [x86_64-linux]
```

No we are ready to run `./configure` script:

```sh
$ ./configure --enable-rubyinterp=yes --prefix=/where/you/want/ruby/to/be/installed/
```

If everything is OK we can go ahead and compile it:

```sh
$ make -j 3
```

`-j` option defines number of processes to make builing faster. It's useful if number of CPU cores is greater than one.

Finally install built Vim to `/where/you/want/ruby/to/be/installed/` directory which you passed as `--prefix` option:

```sh
$ make install
```


## Verify Ruby version

Now let's open your new Vim:

```sh
$ cd /where/you/want/ruby/to/be/installed/
$ ./bin/vim
```

And run the next command:

```vim
:ruby puts RUBY_DESCRIPTION
```

The ouput should looks similar to this:

```
ruby 1.9.3p194 (2012-04-20 revision 35410) [x86_64-linux]
```

So, everything is OK. Enjoy Vim and builtin ruby 1.9 ;)
