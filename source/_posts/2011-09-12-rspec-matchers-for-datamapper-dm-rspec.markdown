---
layout: post
title: "RSpec matchers for DataMapper (dm-rspec)"
date: "2011-09-12 22:25:15"
comments: true
categories: matchers rspec datamapper dm ruby
---

I am currently developing a number of RSpec matchers for DataMapper models testing. You can try it. Just add the next in your Gemfile under `test` group:

    gem 'dm-rspec'

And run `bundle install` to install new gems.

The main idea is to provide the same (almost the same) interface which Shoulda provides to test ActiveRecord models.

At the moment I implemented the next matchers:

* belong\_to
* have\_many
* have\_many\_and\_belong\_to
* have\_property
* have(n).errors\_on(:property)
* have\_many(:association).trough(:another\_association)
* validate\_presence\_of(:property)
* validate\_uniqueness\_of(:property)
* validate\_format\_of(:property).with(/regexp/)


More information you can find on [DataMapper RSpec github page](https://github.com/greyblake/dm-rspec).

Anyone is welcome to join me in developing:)

