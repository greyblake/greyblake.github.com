---
layout: post
title: "Lazy object pattern in ruby"
date: 2014-10-05 21:27
comments: true
categories: ruby lazy object loading
---

I few days ago my colleague [Arthur Shagall](https://github.com/albertosaurus) reviewing my
code suggested me to use **Lazy Object** pattern to postpone some calculations during the load time.
I hadn't heard about the pattern before and even googling it didn't give my much information.
So I have decided to write this article to cover the topic.

## Intention

**Lazy Object** allows you to postpone some calculation until the moment when the actual
result of the calculation is used. That may help you to speed up booting of the application.

## Implementation

It is pretty simple. We create a proxy object that takes a calculation
block as its property and execute it on first method call.

```ruby
class LazyObject < ::BasicObject
  def initialize(&callable)
    @callable = callable
  end

  def __target_object__
    @__target_object__ ||= @callable.call
  end

  def method_missing(method_name, *args, &block)
    __target_object__.send(method_name, *args, &block)
  end
end
```

## Usage example 1

A constant assignment like this:
```ruby
SQUARES = Array.new(10) { |i| i** 2}
```

Could be converted to this one:

```ruby
SQUARES = LazyObject.new { Array.new(10) { |i| i** 2} }
```

So now if you want to use `SQUARES` it still behaves like an array:

```ruby
SQUARES.class  # => Array
SQUARES.size   # => 10
SQUARES        # => [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
```

## Usage example 2

Let's say you have models `State` and `Address` in you Rails application.
What you want do is to validate inclusion of `address.state` in states.

You can just hardcore the list of states:

```ruby
class Address < ::ActiveRecord::Base
  STATES = ["AL", "AK", "AZ", "AR", "CA", "CO"]   # and so on

  validates :state, inclusion: { in: STATES }
end
```
But it does not reflect your changes in DB in any way.

Then you can fetch the values from DB:

```ruby
STATES = State.all.map(&:code)
```

It seems to look better, but there are 2 possible pitfalls:

* It increases load time (1 more SQL query)
* It may cause real troubles if `STATES` is initialized before `State` model is seeded. In this case `STATES` will be empty.

So that is the situation where **Lazy Object** is useful:

```ruby
STATES = LazyObject.new { State.all.map(&:code) }
```


## Ruby gem

If your prefer to have it as a ruby gem,
please take a look at [rubygems.org/gems/lazy_object](http://rubygems.org/gems/lazy_object).



Thanks for reading!
