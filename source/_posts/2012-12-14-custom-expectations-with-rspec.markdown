---
layout: post
title: "Custom expectations with RSpec"
date: 2012-12-14 00:34
comments: true
categories: Ruby RSpec specs test expectation
---

I know you love RSpec's `expect` DSL like this:

```ruby
expect { raise("Boom!") }.to raise_error(RuntimeError, "Boom!")
```

We often write our own custom matchers and I wanna show how is easy to write
custom expectation.


## Desired DSL

Usually when I do things like this I start with DSL. I think it's important since
it must be convenient to use and easy to read.
So turn on your imagination and spend some time on it.

In my example I'm gonna create an expectation to test text written to standard output
and standard error.
There are samples how I wanna use it(desired DSL):

```ruby
# Test text written to standard output
expect { puts "Hello!" }.to write("Hello!")

# Test text written to standard error
expect { warn "Stop it!" }.to write("Stop it!").to(:error)
```

<!--more-->

## Expectation

I bet you've already created number of custom matchers. What about custom expectations?
They are usual custom matches which test blocks of code!

I'm gonna locate the expectation in `spec/support/custom_expecations/write_expecation.rb` file.
I think `spec/support/custom_expecations/` directory is the right place for it since
custom matchers usually are located in `spec/support/custom_matchers/`.

So finally the expectation looks this way:

```ruby spec/support/custom_expecations/write_expecation.rb
RSpec::Matchers.define :write do |message|
  chain(:to) do |io|
    @io = io
  end

  match do |block|
    output =
      case io
      when :output then fake_stdout(&block)
      when :error  then fake_stderr(&block)
      else raise("Allowed values for `to` are :output and :error, got `#{io.inspect}`")
      end
    output.include? message
  end

  description do
    "write \"#{message}\" #{io_name}"
  end

  failure_message_for_should do
    "expected to #{description}"
  end

  failure_message_for_should_not do
    "expected to not #{description}"
  end

  # Fake STDERR and return a string written to it.
  def fake_stderr
    original_stderr = $stderr
    $stderr = StringIO.new
    yield
    $stderr.string
  ensure
    $stderr = original_stderr
  end

  # Fake STDOUT and return a string written to it.
  def fake_stdout
    original_stdout = $stdout
    $stdout = StringIO.new
    yield
    $stdout.string
  ensure
    $stdout = original_stdout
  end

  # default IO is standard output
  def io
    @io ||= :output
  end

  # IO name is used for description message
  def io_name
    {:output => "standard output", :error => "standard error"}[io]
  end
end

```

And that's it. You can try it against the examples from "Desired DSL" section.

Hope that's was useful. I look forward for your comments and suggestions. You know I do! =)
