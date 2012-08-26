---
layout: post
title: "Ruby performance antipatterns"
date: 2012-08-12 13:23
comments: true
published: false
categories: ruby exception performance string concatenation
---

This time I want to describe common ways developers use to slow down ruby applications.

## Exceptions for a control flow

In ruby exceptions are pretty slow.
Consider an example with usage of `respond_to?` and conditional statement
against catching `NoMethodError`:

```ruby
require 'benchmark'

class Obj
  def with_condition
    respond_to?(:mythical_method) ? self.mythical_method : nil
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

Benchmark.bm(15, "rescue/condition") do |x|
  rescue_report     = x.report("rescue:")    { N.times { obj.with_rescue  } }
  condition_report  = x.report("condition:") { N.times { obj.with_if      } }
  [rescue_report / condition_report]
end
```

MRI 1.9.3:
    ruby 1.9.3p194 (2012-04-20 revision 35410) [x86_64-linux]
                            user     system      total        real
    rescue:           111.530000   2.650000 114.180000 (115.837103)
    condition:          2.620000   0.010000   2.630000 (  2.633154)
    rescue/condition:  42.568702 265.000000        NaN ( 43.991767)


MRI 1.8.7 (REE has similar result):
    ruby 1.8.7 (2011-12-28 patchlevel 357) [x86_64-linux]
                            user     system      total        real
    rescue:            80.510000   0.940000  81.450000 ( 81.529022)
    if:                 3.320000   0.000000   3.320000 (  3.330166)
    rescue/condition:  24.250000        inf       -nan ( 24.481970)


As you see the solution with exception is much slower than using conditional statement.
And that's true for all kind of exceptions. Normally exceptions should not be used for a control flow.
Read [Simone Carletti's article](http://www.simonecarletti.com/blog/2010/01/how-slow-are-ruby-exceptions/)
to find out more about exceptions and performance issues.

## Using += to join strings

Avoid using `+=` to concatenate strings in favor of `<<` method.
The result is absolutely the same: add a string to the end of an existing one.
What is the difference then?

See the example:

```ruby
str1 = "first"
str2 = "second"
str1.object_id       # => 16241320

str1 += str2    # str1 = str1 + str2
str1.object_id  # => 16241240, id is changed

str1 << str2
str1.object_id  # => 16241240, id is the same
```

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
  puts "_" * 60 + "\nLENGTH: #{length}"

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
    ____________________________________________________________
    LENGTH: 10
                     user     system      total        real
    +=           0.000000   0.000000   0.000000 (  0.004671)
    <<           0.000000   0.000000   0.000000 (  0.000176)
    += VS <<          NaN        NaN        NaN ( 26.508796)
    ____________________________________________________________
    LENGTH: 100
                     user     system      total        real
    +=           0.020000   0.000000   0.020000 (  0.022995)
    <<           0.000000   0.000000   0.000000 (  0.000226)
    += VS <<          Inf        NaN        NaN (101.845829)
    ____________________________________________________________
    LENGTH: 1000
                     user     system      total        real
    +=           0.270000   0.120000   0.390000 (  0.390888)
    <<           0.000000   0.000000   0.000000 (  0.001730)
    += VS <<          Inf        Inf        NaN (225.920077)
    ____________________________________________________________
    LENGTH: 10000
                     user     system      total        real
    +=           3.660000   1.570000   5.230000 (  5.233861)
    <<           0.000000   0.010000   0.010000 (  0.015099)
    += VS <<          Inf 157.000000        NaN (346.629692)
    ____________________________________________________________
    LENGTH: 100000
                     user     system      total        real
    +=          31.270000  16.990000  48.260000 ( 48.328511)
    <<           0.050000   0.050000   0.100000 (  0.105993)
    += VS <<   625.400000 339.800000        NaN (455.961373)


## Nested iterators

Assume you need to write a function to convert an array into a hash
where keys and values are same as elements of the array:

```ruby
func([1, 2, 3])  # => {1 => 1, 2 => 2, 3 => 3}
```

The next solution would satisfy the requirements:

```ruby
def func(array)
  array.inject({}) { |h, e| h.merge(e => e) }
end
```
And would be extremely slow with big portions of data because it contains
nested methods (`inject` and `merge`), so it's __ O(n<sup>2</sup>) __ algorithm.
But it's obviously that it must be __ O(n) __.
Consider the next:

```ruby
def func(array)
  array.inject({}) { |h, e| h[e] = e; h }
end
```
In this case we do only one iteration over an array without any hard calculation
within the iterator.

See the benchmark:

```
require 'benchmark'

def n_func(array)
  array.inject({}) { |h, e| h[e] = e; h }
end

def n2_func(array)
  array.inject({}) { |h, e| h.merge(e => e) }
end

BASE_SIZE = 10

4.times do |factor|
  size   = BASE_SIZE * (10 ** factor)
  params = (0..size).to_a
  puts "_" * 60 + "\nSIZE: #{size}"
  Benchmark.bm(10) do |x|
    x.report("O(n)" ) { n_func(params)  }
    x.report("O(n2)") { n2_func(params) }
  end
end
```

Output:
    ____________________________________________________________
    SIZE: 10
                    user     system      total        real
    O(n)        0.000000   0.000000   0.000000 (  0.000014)
    O(n2)       0.000000   0.000000   0.000000 (  0.000033)
    ____________________________________________________________
    SIZE: 100
                    user     system      total        real
    O(n)        0.000000   0.000000   0.000000 (  0.000043)
    O(n2)       0.000000   0.000000   0.000000 (  0.001070)
    ____________________________________________________________
    SIZE: 1000
                    user     system      total        real
    O(n)        0.000000   0.000000   0.000000 (  0.000347)
    O(n2)       0.130000   0.000000   0.130000 (  0.127638)
    ____________________________________________________________
    SIZE: 10000
                    user     system      total        real
    O(n)        0.020000   0.000000   0.020000 (  0.019067)
    O(n2)      17.850000   0.080000  17.930000 ( 17.983827)


It's an obvious and trivial example. Just keep in mind that
using nested iterating methods like `each`, `map`, `inject` makes your
algorithm much slower, and it's better to avoid it when it's possible, even
if you code would look less sexy.

If you are not aware about **big O notation** then
[Rob Bell's article](http://rob-bell.net/2009/06/a-beginners-guide-to-big-o-notation/)
will be helpful.
