---
layout: post
title: "Vim preview plugin"
date: "2011-01-23 03:13:14"
comments: true
categories: Vim preview markup plugin markdown textile html
---

## Intro

Preview plugin is a tool developed to help you to preview markup files such as .markdown, .rdoc, .textile, .ronn and .html when you are editing them. It builds html files and opens them in your browser.


## Supported Formats

The plugin supports the next formats:

* markdown(md, mkd, mkdn, mdown) - depends on `bluecloth` ruby gem
* rdoc - depends on `github-markup` ruby gem
* textile - depends on `RedCloth` ruby gem
* html(htm)
* ronn - depends on `ronn` ruby gem

<!--more-->

## Dependencies

The plugin requires a builtin ruby interpreter. It means that your Vim
should be compiled with `--enable-rubyinterp` option.
To find out does you Vim have builtin ruby interpreter you can do the next:
    :echo has('ruby')

If output is `1` the ruby interpreter is builtin.

The second thing you should verify is that you have installed all necessary
ruby gems. Please see "Supported Formats" section to find out what gems you need.


## Installation

At first you should get the plugin. You can clone my repository from github like this:

    git clone git://github.com/greyblake/vim-preview.git

Or download it from [vim.org](http://www.vim.org/scripts/script.php?script_id=3344)

To install the plugin just copy `autoload`, `plugin`, `doc` directories into your `.vim` directory:

    cd vim-preview
    cp ./autoload ~/.vim -R
    cp ./plugin ~/.vim -R
    cp ./doc ~/.vim -R

If you use [Pathogen](http://www.vim.org/scripts/script.php?script_id=2332) just unzip/clone the plugin to `~/.vim/bundle` directory.
If don't use  [Pathogen](http://www.vim.org/scripts/script.php?script_id=2332), I would recommend you to start.

## Usage

* `<Leader>P` - will open current file converted to HTML in your browser.

I want to remind that `<Leader>` in most cases is `\` key.

To get more information about the plugin and how it can be customized read help in Vim editor:

    :help Preview

## If you found a bug ...

Report it me!

