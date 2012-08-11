---
layout: post
title: "Unexpected Ruby behaviour"
date: 2012-08-10 23:28
comments: true
categories: unexpected ruby behaviour
---


Ruby is a cool language with intuitive grammar. However there are a number of things which don't seem to be expected.
It might take long hours to debug some weird issues for unenlightened newbies.



## Implicitly variable declaration

Variable mentioned in conditional block of code become declared and initialized with `nil` even if the declaration was not executed.

```ruby
if false
  var = "never executed"
end
var # => nil
```

**Expected:** addressing to `var` raises `NameError: undefined local variable or method 'var'`

## Calling #utc and #gmt on Time object removes time zone information

```ruby
t = Time.new  # => Sat Aug 11 01:11:52 +0300 2012
t.utc         # => Fri Aug 10 22:11:52 UTC 2012
t             # => Fri Aug 10 22:11:52 UTC 2012, WTF?

t = Time.new  # => Sat Aug 11 01:17:06 +0300 2012
t.gmtime      # => Fri Aug 10 22:17:06 UTC 2012
t             # => Fri Aug 10 22:17:06 UTC 2012, WTF?
```

IMHO, this methods should be called `utc!` and `gmtime!` instead.
I have an experience when it caused a really voodoo thing: a test failed only from 20:00 to 00:00
in USA on CI server, and could never be reproduced in my time zone.

Instead it's better to use `getutc` and `getgm` methods which return UTC and GMT time accordingly,
but don't change Time object:

```ruby
t = Time.new  # => Sat Aug 11 01:11:52 +0300 2012
t.getutc      # => Fri Aug 10 22:11:52 UTC 2012
t             # => Sat Aug 11 01:11:52 +0300 2012

t = Time.new  # => Sat Aug 11 01:17:06 +0300 2012
t.getgm       # => Fri Aug 10 22:17:06 UTC 2012
t             # => Sat Aug 11 01:17:06 +0300 2012
```



## Methods don't return value from ensure statement

Usually ruby methods return the value of the last method line unless `return` is called explicitly.
But how about this?


```ruby
def run
  1
ensure
  puts "ensure block..."
  2
end

run # => 1
# pints `ensure block...`
```


So if you want to return a value from `ensure` statement use `return` word:

```ruby
def run
  1
ensure
  return 2
end

run # => 2
```

## lambda and Proc.new acts differently

It's a well known thing but I want to remind.
There 2 differences between proc objects created with `lambda` and `Proc.new`:

* `lambda` raises `ArgumentError` if parameter is missing when `Proc.new` uses `nil` instead.

```ruby
lm = lambda   {|a, b| "#{a.inspect} and #{b.inspect}" }
pr = Proc.new {|a, b| "#{a.inspect} and #{b.inspect}" }

lm.call(10)  # => ArgumentError: wrong number of arguments (1 for 2)
pr.call(10)  # => "10 and nil"
```

* For `lambda` word `return` means returning from proc object, when for `Proc.new` it means returning from scope where proc is defined.

```ruby
def lambda_method
  lm = lambda { return 10 }     # return from lambda
  half = lm.call
  half * 2
end

def proc_method
  pr = Proc.new { return 10 }   # return from proc_method
  half = pr.call
  half * 2
end

lambda_method   # => 20
proc_method     # => 10
```

Note there is also method `proc`. In Ruby 1.8 it's a synonym for `lambda`
but in Ruby 1.9 it's a synonym for `Proc.new`. So avoid using `proc` to keep you code compatible
for both ruby versions.


## DelegateClass instance doesn't eql itself

Ruby standard library provides [DelegateClass](http://www.ruby-doc.org/stdlib-1.9.3/libdoc/delegate/rdoc/Object.html)
which can be [pretty useful](http://pivotallabs.com/users/jdean/blog/articles/1138-delegateclass-rocks-my-world).

```ruby
require 'delegate'

class Animal
end

class Dog < DelegateClass(Animal)
end


animal = Animal.new
dog = Dog.new(animal)

dog.eql?(dog)  # => false
```

It happens because `eql?` is delegated to base object(animal):

```ruby
dog.eql?(animal)  # => true
```

On other hand `equal?` is not:

```ruby
dog.equal?(dog)     # => true
dog.equal?(animal)  # => false
```

