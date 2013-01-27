---
layout: post
title: "Powering less to highlight syntax and display line numbers"
date: "2011-09-23 17:38:35"
comments: true
categories: less syntax-highlight syntax unix linux bash
---

Since I am a command line guy I use __less__ tool everywhere and everytime to quickly take a look at files. And quite often those files are different scripts and source code. So, it would be great if syntax was highlighted automatically when I open a file with less. And probably it would be great as well if I saw line numbers.

<!--more-->

## Making less highlight syntax

It didn't take a long time to find a solution.  There is a pretty tool called __source-highlight__.

It might be exist in standard repository of your system. In Debian like Linux distribution(Debian, Ubuntu, Mint, etc) you can install it with command:

    apt-get install source-highlight

It supports a realy big list of languages(in my case 143) and 12 output formats(esc, html, javadoc, latex, etc). You can use man to get closer with it:

    man source-highlight

__source-highlight__ package provides a little script which receives a source code file and prints highlighted code. Its path is `/usr/share/source-highlight/src-hilite-lesspipe.sh `. So we're gonna to use it with __less__.

Fortunately __less_ is require flexible tool and we can customize it with number of invironment variables. One of them is `LESSOPEN`. A quote from `man less`:

    LESSOPEN
        Command line to invoke the (optional) input-preprocessor.

So we can you any pipes to prepare an input for __less__.

Another variable we are going to use is `LESS`:

    LESS
        Options which are passed to less automatically.

We will use `-R` option to allow ANSI colors.

So just set up this to variables:

```bash
export LESSOPEN="| /usr/share/source-highlight/src-hilite-lesspipe.sh %s"
export LESS=' -R '
```

## Making less display line numbers

To display line numbers just use `-N` option. If you want __less__ to use it everytime add it to `LESS` variable as well:

```bash
export LESS=' -RN '
```

## Make it work everytime

To make it work in all shell session just add the next code to the end of you `$HOME/.bashrc` file:

```bash

# syntax highlight for less
export LESSOPEN="| /usr/share/source-highlight/src-hilite-lesspipe.sh %s"
export LESS=' -R -N'

```

Hope this was useful for you. Don't be shy and leave you comments.

