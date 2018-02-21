---
layout: post
title: "How to fix intermittent test failures"
date: 2018-02-19 20:18
comments: true
categories: tests, ci, ruby, rust, rspec, rails
---

![Red build in CI? Works on my machine](/images/how-to-fix-intermittent-test-failures.jpg).

You probably happened to face some nasty tests in your continuous integration,
that fails from time to time and make your build red.
It slows down the deployment pipeline and could be very annoying.

In my opinion, intermittent tests could be divided into two major groups: order dependent tests and intermittent tests by themselves.
I will cover both in this article.

* [Order dependent tests](/blog/2018/02/19/how-to-fix-intermittent-test-failures/#order-dependent-tests)
  * [How to reproduce order dependent tests?](/blog/2018/02/19/how-to-fix-intermittent-test-failures/#how-to-reproduce-order-dependent-tests)
  * [How to fix order dependent tests?](/blog/2018/02/19/how-to-fix-intermittent-test-failures/#how-to-fix-order-dependent-tests)
* [Single intermittent tests](/blog/2018/02/19/how-to-fix-intermittent-test-failures/#single-intermittent-tests)
  * [Order related problems](/blog/2018/02/19/how-to-fix-intermittent-test-failures/#order-related-problems)
    * [Database selection without ordering](/blog/2018/02/19/how-to-fix-intermittent-test-failures/#database-selection-without-ordering)
    * [Unstable sort](/blog/2018/02/19/how-to-fix-intermittent-test-failures/#unstable-sort)
    * [Iterating over HashMap-like structures](/blog/2018/02/19/how-to-fix-intermittent-test-failures/#iterating-over-hashmap-like-structures)
    * [Other order related problems](/blog/2018/02/19/how-to-fix-intermittent-test-failures/#other-order-related-problems)
  * [Time and timezone related failures](/blog/2018/02/19/how-to-fix-intermittent-test-failures/#time-and-timezone-related-failures)
* [Conclusion](/blog/2018/02/19/how-to-fix-intermittent-test-failures/#confusion)
* [Resources](/blog/2018/02/19/how-to-fix-intermittent-test-failures/#resources)

## <a name="order-dependent-tests"></a> Order dependent tests

**An order dependent test is a test that always passes in isolation but fails when it runs with other tests in a particular order**.

Example: let's say we have tests `A` and `B`. Test `A` passes in isolation and passes
when we run sequence `A, B`, but permanently fails in sequence `B, A`.

Usually, it happens when test `B` does not clean up the environment properly and this in some way affects `A`.


### <a name="how-to-reproduce-order-dependent-tests"></a> How to reproduce order dependent tests?

Quite often in CI tests run in a random order. For reproduction, it's important to know the exact order.

Let's say you know from your CI logs that in a test sequence `A, B, C, D, E, F`  test `E` fails.
Most likely it fails because one of the preceding tests changes the global environment.

Try to run sequence `A, B, C, D, E` locally to confirm the hypothesis.
Make sure that tests run exactly in the specified order. If you're using RSpec you need to
pass `--order defined` option.

    rspec --order defined ./a_spec.rb ./b_spec.rb ./c_spec.rb ./d_spec.rb ./e_spec.rb

Now how do you know which of `A`, `B`, `C`, `D` makes `E` break? You need to experiment, running different sequences like
`A, E`, `B, E`, `C, E`, `D, E`. If there are a lot of tests it may take long, so I prefer to use binary search.
Split preceding tests into 2 groups: `A, B` and `C, D` and determine which of the following sequences fail:
`A, B, E` or `C, D, E`. Then do the same with the failing group until you get a minimal reproducible example. E.g.

    rspec --order defined ./b_spec.rb ./e_spec.rb

UPDATE: As few of my readers pointed, there is [rspec --bisect](https://relishapp.com/rspec/rspec-core/docs/command-line/bisect)
that does it already automatically. Thanks!


<!--more-->

### <a name="how-to-fix-order-dependent-tests"></a> How to fix order dependent tests?

Now you need to inspect test `B` to see where exactly it doesn't clean up the environment.
Quite often it can be one of the following reasons:

* The test creates new records in the database without deleting them after.
* The test stubs some object methods (e.g. `Time.now`) without reverting the change.

It often happens with `Timecop`:
```ruby
before { Timecop.freeze(2017, 02, 17) }

# If this is forgotten, the time will be frozen for all the subsequent tests
after { Timecop.return }
```

* The test creates files in the file system and does not delete them after.
* The test defines some classes that conflict with real classes from the code and it breaks autoloading mechanism in Rails.

The latest point may not be easy to understand, so let me illustrate it with an example.
Let's say we have `DummyModule` module that we want to test:

```ruby
module DummyModule
  def dummy
    "dummy"
  end
end
```

The test may look like the following:

```ruby
describe DummyModule do
  class DummyService
    include DummyModule
  end

  subject(:service) { DummyService.new }

  it "includes dummy method" do
    expect(service.dummy).to eq "dummy"
  end
end
```

So what's wrong with it? It creates a new global constant named `DummyService`.
The constant lives even when the test ends. If you define `DummyService` in multiple tests they overlap
and may have side effects. Or if you have real `DummyService` class in you rails app in `app/services/dummy_service.rb`,
and you run a test sequence, where `dummy_service_spec.rb` follows `dummy_module_spec.rb`, you may get an order
dependent test.

Since `DummyService` is already defined in `dummy_module_spec.rb` rails autoload will never try to load
`app/service/dummy_service.rb` file and as result `dummy_service_spec.rb` will fail, because
it tests a wrong version of `DummyService` (defined in `dummy_module_spec.rb`).

To test such modules you should prefer to use anonymous classes:

```ruby
describe DummyModule do
  subject(:service) { dummy_class.new }

  let(:dummy_class) do
    Class.new do
      include DummyModule
    end
  end

  it "includes dummy method" do
    expect(service.dummy).to eq "dummy"
  end
end
```

Such test does not pollute global environment.


## <a name="single-intermittent-tests"></a> Single intermittent tests

### <a name="order-related-problems"></a> Order related problems

Sometimes programming languages and databases have undefined behavior regarding
order related operations. We, developers, may make wrong assumptions about it
and introduce a bug or an intermittent test.
Fortunately, those issues are often easy to spot and fix.

#### <a name="database-selection-without-ordering"></a> Database selection without ordering

Most of the databases do not guarantee an order of returned items unless it's explicitly specified in the request.
You should always keep this in mind if your test relies on a specific order.

Assume we have an ActiveRecord model `User` and we want to write a test for
`fetch_all_users` function, which returns all existing records from the database.

```ruby
def fetch_all_users
  User.all
end

it "fethes all existing records" do
  User.create!(name: "Anthony")
  User.create!(name: "Ahmed")
  User.create!(name: "Paulo")
  User.create!(name: "Max")
  User.create!(name: "Ricardo")

  names = fetch_all_users.map(&:name)
  expect(names).to eq ["Anthony", "Ahmed", "Paulo", "Max", "Ricardo"]
end
```

At first glance, this test may look innocent. And it will probably pass if you try to run it.
I had to loop the test and run it about 5000 times to reproduce one single failure
(with PostgreSQL 9.5, and RSpec option `use_transactional_fixtures` set to `false`):

    1) fethes all existing records
       Failure/Error: expect(names).to eq ["Anthony", "Ahmed", "Paulo", "Max", "Ricardo"]

         expected: ["Anthony", "Ahmed", "Paulo", "Max", "Ricardo"]
              got: ["Max", "Ricardo", "Anthony", "Ahmed", "Paulo"]


There are two possible solutions to make the test stable.

First one is to modify `fetch_all_users` to enforce the order of returned items:
```ruby
def fetch_all_users
  User.all.order(:id)
end
```

The second one, if you really don't care about the order,
is to update the test to be order-agnostic.
With RSpec you can use [contain_exactly](http://www.rubydoc.info/gems/rspec-expectations/RSpec/Matchers#contain_exactly-instance_method)
matcher for that. As the documentation says:

{% blockquote %}
Passes if actual contains all of the expected regardless of order.
{% endblockquote %}

So the expectation statement becomes:

```ruby
  expect(names).to contain_exactly("Anthony", "Ahmed", "Paulo", "Max", "Ricardo")
```


#### <a name="unstable-sort"></a> Unstable sort

You should learn the difference between stable and unstable sorting algorithms and know which one
is used by default in your programming language and your database.

Let's take a look at an example with a stable sorting algorithm.
Here we have 3 people, 2 of them have the same age. We're gonna sort people by age.

```ruby
# Reproduced on MRI 2.4.2 which has a stable sorting algorithm.
# NOTE: different ruby implementation and versions use different sort algorithms be default.

class Person
  attr_reader :name, :age

  def initialize(name, age)
    @name = name
    @age = age
  end

  def <=> (other)
    self.age <=> other.age
  end
end

people_set1 = [
  Person.new("Bernard", 40),
  Person.new("Johannes", 30),
  Person.new("Steffen", 40)
]

# Bernard precedes Steffen (as in the input data set)
p people_set1.sort.map(&:name) # => ["Johannes", "Bernard", "Steffen"]

# Now let's swap Bernard and Steffen
people_set2 = [
  Person.new("Steffen", 40),
  Person.new("Johannes", 30),
  Person.new("Bernard", 40)
]

# Now Steffen precedes Bernard
p people_set2.sort.map(&:name) # => ["Johannes", "Steffen", "Bernard"]
```

**Stable sorting algorithms retain the relative order of items with equal keys**.

As you may conclude, unstable sorting algorithms are those, that do not match the
definition of "stable sorting algorithm".

However, there are 2 possible types of unstable sorting algorithms:

* Those that persist the same output for the same given input
* Those that may return different output when the same input is given

The second is not desired and must be avoided since it introduces
a real randomness. An example could be a [quicksort](https://en.wikipedia.org/wiki/Quicksort)
implementation with literally randomly chosen pivot.

Most of the languages have the first type of unstable sort. But it's good to be on the alert.

By the way, if you wonder what kind of sorting algorithm has your Ruby version,
I recommend you to take a look at this [stackoverflow answer](https://stackoverflow.com/a/44486562/1013173).


#### <a name="iterating-over-hashmap-like-structures"></a> Iterating over HashMap-like structures

HashMap-like structures are widely used in many scripting languages: in Ruby it's called "hash", in JavaScript - "object",
in Python - "dictionary", in PHP - "associated array", etc.

The problem is, some implementations do not guarantee order persistence on iteration over HashMap keys.
For example it was the case for Ruby before version 1.9, that's why ActiveSupport used to have
[OrderedHash](http://api.rubyonrails.org/v3.2/classes/ActiveSupport/OrderedHash.html).

In case of JavaScript [the traversion order was only defined in ES6](http://2ality.com/2015/10/property-traversal-order-es6.html).


Here is a little Rust program, that illustrates the issue with an equivalent Ruby code in
the comments.

```rust
// Reproduced with rust version 1.22.1
use std::collections::HashMap;

fn main() {
    // hash = { 1 => 1, 2 => 4 }
    let mut hash = HashMap::new();
    hash.insert(1, 1);
    hash.insert(2, 4);

    // keys = hash.keys
    let keys: Vec<i32> = hash.keys().map(|x| *x).collect();

    // expect(keys).to eq [1, 2]
    assert_eq!(keys, vec![1, 2])
}

```

If you run this program multiple times, sometimes it may succeed, sometimes it fails:

    thread 'main' panicked at 'assertion failed: `(left == right)`
      left: `[2, 1]`,
     right: `[1, 2]`', src/main.rs:16:4


The solution is the same as for the previous order related problems.
Either to update the code to sort keys explicitly or to make the test be order agnostic:

```rust
// keys = hash.keys
let mut keys: Vec<i32> = hash.keys().map(|x| *x).collect();

// keys.sort!
keys.sort();

// expect(keys).to eq [1, 2]
assert_eq!(keys, vec![1, 2])
```

#### <a name="other-order-related-problems"></a> Other order related problems

Everything that has not 100% defined behavior may lead to the similar issues.
There are few other examples:

* Iterating over entries in the file system may vary depending on a file system, operating system, file system drivers, etc.
* If you run concurrent operations they are not guaranteed to finish in the order they start. So you may want to do some kind of sorting to aggregate the final result.

### <a name="time-and-timezone-related-failures"></a> Time and timezone related failures

Tests should not depend on current time and date.

It's not obvious, but sometime a test may fail on CI just because it runs in a specific time in a
specific (different from your local) timezone. E.g. it may fail in time frame from 20:00 to 00:00
in CI server that runs in Pacific Time Zone, but the failure may not be reproducible in Europe.

If you suspect this, the first step would be to change your local time settings in order to
reproduce the same time conditions as on the CI server, when the test failed.

After you're able to reproduce the failure locally it must be relatively easy to debug.

Another example of a test that depends on the current time:

```ruby
def current_year
  Time.now.year
end

it "returns current year" do
  expect(current_year).to eq 2018
end
```

Obviously, on the 1st of January 2019 it will start failing.
For this test you'd need to stub the current time with [Timecop](https://github.com/travisjeffery/timecop):

```ruby
it "returns current year" do
  Timecop.freeze(2018, 2, 19) do
    expect(current_year).to eq 2018
  end
end
```

## <a name="confusion"></a> Conclusion

We have covered the most common cases where an intermittent test can be
introduced to a smooth CI process. However, some situations may be tricker and tests may fail
only when multiple of the covered factors combined together.

Usually, it is better to spot problems on the code review stage,
at least by now you should know what you should pay attention to.

Also it worth saying, that the article does not cover problems related to concurrency and asynchronous
communication which are very big topics by themselves.

Thanks for reading please give me feedback.
What was the toughest intermittent you had to debug? =)


## <a name="resources"></a> Resources

* [Avoiding intermittent test failures](https://developer.mozilla.org/en-US/docs/Mozilla/QA/Avoiding_intermittent_oranges)
* [Stable Sorting in Ruby](https://8thlight.com/blog/will-warner/2013/03/26/stable-sorting-in-ruby.html)
* [Comparison of different ruby implementations for sorting stability](https://stackoverflow.com/a/44486562/1013173)
