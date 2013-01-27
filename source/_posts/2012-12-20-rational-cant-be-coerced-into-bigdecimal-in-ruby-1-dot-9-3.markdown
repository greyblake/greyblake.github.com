---
layout: post
title: "Rational can't be coerced into BigDecimal in ruby 1.9.3"
date: 2012-12-20 16:34
comments: true
categories:  ruby ruby1.9 coercion rational BigDecimal
---

Trying to move a rails application from ruby 1.8.7 to 1.9.3 I ran into coercion
issue of `Rational` class.

Ruby 1.9.3:
```ruby
require 'bigdecimal'
require 'rational'

# You can multiply Rational against BigDecimal
Rational(1) * BigDecimal('1')  # => <BigDecimal:a566d0,'0.1E1',9(36)>

# But you can't do the same when you change order
BigDecimal('1') * Rational(1)  # => TypeError: Rational can't be coerced into BigDecimal
```

On other hand in Ruby 1.8.7:

```ruby
require 'bigdecimal'
require 'rational'

# BigDecimal * Rational works OK
BigDecimal('1') * Rational(1)  # => 1.0 (Float)

# But Rational * BigDecimal doesn't
Rational(1) * BigDecimal('1')  # => TypeError: Rational can't be coerced into BigDecimal
```

It's looks weird. So I can only say for sure that `Rational` -> `BigDecimal`
coercion is not implemented in Ruby.

I've tried to fix it with simple monkey patch:

```ruby
# Works only for Ruby1.9.3
class Rational
  def coerce(value)
    case value
    when BigDecimal
      return self, value
    else
      super
    end
  end
end
```

And it works OK against simple examples:

```ruby
Rational(1) * BigDecimal('1')  # => <BigDecimal:a566d0,'0.1E1',9(36)>
BigDecimal('1') * Rational(1)  # => <BigDecimal:a566d0,'0.1E1',9(36)>
```

But it causes intermittent segmentation faults when I run Rails application.

Any ideas about the coercion? Is it expected behaviour of ruby1.9.3?
I found no bugs reported this issue on
[https://bugs.ruby-lang.org](bugs.ruby-lang.org).

I'll appreciate any feedback. Thanks.
