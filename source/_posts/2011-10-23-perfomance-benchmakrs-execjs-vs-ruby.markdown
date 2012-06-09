---
layout: post
title: "Perfomance benchmakrs ExecJS VS Ruby"
date: "2011-10-23 08:49:33"
comments: true
categories: ExecJS javascript benchmark perfomance ruby
---

Yesterday on RubyShift [Thorben Schröder](http://www.xing.com/profile/Thorben_Schroeder) talked about ExecJS and using it for validation following DRY principle: you implement only JavaScript validator and then use it in Ruby code as well.  Sounds great, right? He provided interesting benchmark results which really suprised me. ExecJS can be few times faster than Ruby 1.9.2 if you're are using [therubyracer](https://github.com/cowboyd/therubyracer)(wrapper for Google's V8). I decided to make my own benchmarks  There is trivial example - function which caclucates fibonacci numbers. I know the algorithm sucks, but my goal is to compare perfomance.

```ruby benchmark.rb

require 'rubygems'
require 'execjs'
require 'benchmark'

js =<<JAVASCRIPT
function fib(a){
  if(a < 2)
    return a;
  return fib(a-1) + fib(a-2);
}
JAVASCRIPT
context = ExecJS.compile(js)

def fib(a)
  return a if a < 2
  fib(a-1) + fib(a-2)
end

[1, 10, 20, 30, 40].each do |n|
  puts "N = #{n}"
  Benchmark.bm(7) do |x|
    x.report("Ruby:") { fib(n) }
    x.report("ExecJS:") { context.call("fib", n) }
  end
  puts '-' * 52
end

```


Here is the output:



    N = 1
                 user     system      total        real
    Ruby:    0.000000   0.000000   0.000000 (  0.000008)
    ExecJS:  0.000000   0.000000   0.000000 (  0.000318)
    ----------------------------------------------------
    N = 10
                 user     system      total        real
    Ruby:    0.000000   0.000000   0.000000 (  0.000026)
    ExecJS:  0.000000   0.000000   0.000000 (  0.000156)
    ----------------------------------------------------
    N = 20
                 user     system      total        real
    Ruby:    0.000000   0.000000   0.000000 (  0.002585)
    ExecJS:  0.000000   0.000000   0.000000 (  0.000550)
    ----------------------------------------------------
    N = 30
                 user     system      total        real
    Ruby:    0.310000   0.000000   0.310000 (  0.312819)
    ExecJS:  0.040000   0.000000   0.040000 (  0.043603)
    ----------------------------------------------------
    N = 40
                 user     system      total        real
    Ruby:   39.780000   0.050000  39.830000 ( 39.955098)
    ExecJS:  3.940000   0.020000   3.960000 (  3.943733)
    ----------------------------------------------------

Looks like ExecJS needs some time for "initializing" and there is no reason to use it for small calculation. But it can be ten times faster in case of long time computing!

I would not decide to use ExecJS for validation but however the benchmark results were amazing, and we've got to keep in mind that ExecJS can provide such perfomance benefits.

