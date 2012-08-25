---
layout: post
title: "Ruby performance antipatterns"
date: 2012-08-12 13:23
comments: true
published: false
categories: ruby exception performance string concatenation
---

This time I want to describe common ways developers use to slow down ruby applications.

<!--Prior to continue reading make sure your know what [sarcasm](http://en.wikipedia.org/wiki/Sarcasm) is.-->

## Using exceptions for a control flow instead of conditional statements

In ruby exceptions are pretty slow.
Consider an example with usage of `respond_to?` against catching `NoMethodError`:

```ruby
require 'benchmark'

class Obj
  def with_if
    if respond_to? :mythical_method
      self.mythical_method
    else
      nil
    end
  end

  def with_rescue
    self.mythical_method
  rescue NoMethodError
    nil
  end
end

obj = Obj.new
N = 10_000_000

puts RUBY_DESCRIPTION

Benchmark.bm(15, "rescue / if") do |x|
  rescue_report = x.report("rescue:")  { N.times { obj.with_rescue  } }
  if_report     = x.report("if:")      { N.times { obj.with_if      } }
  [rescue_report / if_report]
end
```

MRI 1.9.3:
    ruby 1.9.3p194 (2012-04-20 revision 35410) [x86_64-linux]
                          user     system      total        real
    rescue:         111.530000   2.650000 114.180000 (115.837103)
    if:               2.620000   0.010000   2.630000 (  2.633154)
    rescue / if      42.568702 265.000000        NaN ( 43.991767)


MRI 1.8.7 (REE has similar result):
    ruby 1.8.7 (2011-12-28 patchlevel 357) [x86_64-linux]
                         user     system      total        real
    rescue:         80.510000   0.940000  81.450000 ( 81.529022)
    if:              3.320000   0.000000   3.320000 (  3.330166)
    rescue / if     24.250000        inf       -nan ( 24.481970)


As you see catching `NoMethodError` are much slower than using `respond_to?` with `if` statement.
And that's true for all kind of exceptions. Normally exceptions should not be used for a control flow.
Read [Simone Carletti's article](http://www.simonecarletti.com/blog/2010/01/how-slow-are-ruby-exceptions/)
to find out more about exceptions and performance issues.

## Using += to join strings

Avoid using `+=` to concatenate strings in favor of `<<` method.
The result is absolutely the same: add a string to the end of an existing one.
What is the difference then?

See an example:

```ruby
str1 = "first"
str2 = "second"
str1.object_id       # => 16241320

str1 += str2
str1.object_id  # => 16241240, id is changed

str1 << str2
str1.object_id  # => 16241240, id is the same
```

Note that `str1 += str2` equals to `str1 = str1 + str2`.

When you use `+=` ruby creates a temporal object which is result of `str1 + str2`.
Then it overrides `str1` variable with reference to the new built object.
On other hand `<<` modifies existing one.

As result of using `+=` you have the next disadvantages:

* More calculation to join strings.
* Redundant string object in memory (previous value of `str1`), which approximates time when
[GC](http://en.wikipedia.org/wiki/Garbage_collection_%28computer_science%29) will trigger.


How `+=` is slow? Basically it depends on length of strings you have operation with.

```ruby
require 'benchmark'

N = 1000
BASIC_LENGTH = 10

5.times do |factor|
  length = BASIC_LENGTH * (10 ** factor)
  puts "LENGTH: #{length}"

  Benchmark.bm(10, '+= VS <<') do |x|
    concat_report = x.report("+=")  do
      str1 = ""
      str2 = "s" * length
      N.times { str1 += str2 }
    end

    modify_report = x.report("<<")  do
      str1 = "s"
      str2 = "s" * length
      N.times { str1 << str2 }
    end

    [concat_report / modify_report]
  end
end
```

Output:
    LENGTH: 10
                    user     system      total        real
    +=          0.000000   0.010000   0.010000 (  0.003917)
    <<          0.000000   0.000000   0.000000 (  0.000178)
    += VS <<         NaN        Inf        NaN ( 22.025469)
    LENGTH: 100
                    user     system      total        real
    +=          0.030000   0.000000   0.030000 (  0.028815)
    <<          0.000000   0.000000   0.000000 (  0.000214)
    += VS <<         Inf        NaN        NaN (134.888393)
    LENGTH: 1000
                    user     system      total        real
    +=          0.250000   0.110000   0.360000 (  0.361804)
    <<          0.000000   0.000000   0.000000 (  0.001023)
    += VS <<         Inf        Inf        NaN (353.568966)
    LENGTH: 10000
                    user     system      total        real
    +=          3.310000   1.460000   4.770000 (  4.788742)
    <<          0.010000   0.000000   0.010000 (  0.012822)
    += VS <<  331.000000        Inf        NaN (373.474173)
    LENGTH: 100000
                    user     system      total        real
    +=         32.070000  15.230000  47.300000 ( 47.522805)
    <<          0.050000   0.040000   0.090000 (  0.104994)
    += VS <<  641.400000 380.750000        NaN (452.623754)

