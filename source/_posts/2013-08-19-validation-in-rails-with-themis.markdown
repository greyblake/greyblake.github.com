---
layout: post
title: "Validation in rails with Themis"
date: 2013-08-19 17:21
comments: true
categories:  Rails, ActiveRecord, validation, ruby
---

Sometimes ActiveRecord is not enough to meet complicated validation needs.
At [TMXCredit](http://tmxcredit.com/) we've created [Themis](https://github.com/TMXCredit/themis) -
ActiveRecord extension which helps to organize validations in a better way and adds
some flexibility. Here I'm gonna describe some problems which Themis solves after that
I'll take a brief look at possible alternative solutions.


## Modular validation

Themis allows you to extract duplicated validation into model.
Usually rails applications are small enough so you don't need it. But sometimes
you do.


The next example is pretty flat(in real life you probably would use STI or composition to
represent `Doctor` and `Patient` models) but it illustrates where Themis could be useful.

Let's say you have 2 models:


```ruby
class Doctor < ActiveRecord::Base
  validates :first_name, :last_name, :email, :diploma,
            :presence => true
end

class Patient < ActiveRecord::Base
  validates :first_name, :last_name, :email, :age,
            :presence => true
end
```

You see that both models have same validation of `first_name`, `last_name` and `email`.

Themis allows to fix the duplication problem by extracting common validations into
a module:

```ruby
# Module with common validations.
module PersonValidation
  extend Themis::Validation

  validates :first_name, :last_name, :email, :presence => true
end

class Doctor < ActiveRecord::Base
  # import validation of first_name, last_name, email
  include PersonValidation

  validates :diploma, :presence => true
end

class Patient < ActiveRecord::Base
  include PersonValidation

  validates :age, :presence => true
end
```
<!--more-->

So now we keep common validation in one place.
If you want you can include validation modules into each other to combine
necessary validation.

## Validation scenarios

Here is another problem which Themis solves.

We have the following models:


```ruby
class User < ActiveRecord::Base
  has_one :person
  has_many :user_accounts
end

class Person < ActiveRecord::Base
  attr_accessible :first_name, :last_name, :birhday

  belongs_to :user
end

class UserAccount < ActiveRecord::Base
  attr_accessible :email, :login

  belongs_to :user
end
```

So we have model graph like this with `User` model on the top:

![Themis - model graph](http://i1078.photobucket.com/albums/w484/greyblake/themis_model_graph.png)

it's pretty small, but in real life the graph can be much deeper.

What would you do if you needed to apply different validations depending on context?
For example according to your business requirements users must be allowed to use
your application only in case if they filled in all of the fields.
So you need to validate presense of `first_name`, `last_name` and `birhday`
on `Person` model and `email`, `login` and `password` on `UserAccount`.

It's not a problem, just add the validations to appropriate models:

```ruby
class Person < ActiveRecord::Base
  validates :first_name, :layout, :birhday, :presence => true
end

class UserAccount < ActiveRecord::Base
  validates :email, :login, :presence => true
end
```

There are some percent of users who don't finish registration process.
But your marketing department wants to have an ability to contact them
in case if they have entered an email address.

So that's where the issue is: you can't save records using validation rules written above.

With Themis can declare number of validation strategies
and depending on context chose which one you need.

Here is how complete solution looks:

```ruby
class User < ActiveRecord::Base
  has_one :person
  has_many :user_accounts

  accepts_nested_attributes_for :person, :user_accounts

  # Declare validations. Use :full as default.
  has_validation :full, :default => true
  has_validation :partial
end

class Person < ActiveRecord::Base
  attr_accessible :first_name, :last_name, :birhday

  belongs_to :user

  # Declare full validation
  has_validation :full do |model|
    model.validates :first_name, :last_name, :birhday, :presence => true
  end

  # Delcare partial validation. Nothing to validate.
  has_validation :partial
end

class UserAccount < ActiveRecord::Base
  attr_accessible :email, :login

  has_validation :full do |model|
    model.validates :login, :email, :presence => true
  end

  has_validation :partial do |model|
    model.validates :email, :presence => true
  end
end
```

And here how you would use it somewhere in controller:

```ruby
# Create model initialized with params
user = User.new(
  :person => {
    :first_name => "Alex",
    :last_name  => "DeLarge",
    :birhday    => "1962"
  },
  :user_accounts => [{
    :email => "clockwork@orange.com"
  }]
)

user.valid? # => false, because login is missing

# Try to apply partial validation
user.use_validation(:partial)
user.valid? # => true

# We can save it
user.save!
```

## Alternative solutions

If you think Themis is overkill for your project, you still have some options.

### Using ActiveSupport::Concern for modularity

`ActiveSupport::Concern` is another way which allows to extract common validations
into module. Here how would `PersonValidation` module described above could look:

```ruby
module PersonValidation
  extend ActiveSupport::Concern

  included do
    validates :first_name, :last_name, :email, :presence => true
  end
end

```

### Using conditional validation

If your requirements aren't so fancy you can be satisfied with simple
conditional validation, e. g.

```ruby
class Person < ActiveRecord::Base
  validates :first_name, :last_name, :birhday, :presence => true,
            :if => :use_full_validation?

  # Lets add one more validation statement(for the next example)
  validates :first_name, :last_name, :length => { :maximum => 255 },
            :if => :use_full_validation?

  def use_full_validation?
    # Some logic goes here
  end
end
```

To DRY up `:if` options it's good to use `with_options` method:


```ruby
class Person < ActiveRecord::Base
  with_options :if => :use_full_validation? do |person|
    person.validates :first_name, :last_name, :birhday, :presence => true
    person.validates :first_name, :last_name, :length => { :maximum => 255 }
  end
end
```

Now `:if => :use_full_validation` will be additonaly passed to every method call
on `person` inside the block.

### Vanguard

Guys from [ROM project](http://rom-rb.org/) have own validator called
[Vanguard](https://github.com/mbj/vanguard)(previous name is Aqeuitas).
The sweet thing about it is that it allows to seperate validations and
models according to DataMapper approach. The downside is if you use ActiveRecord
you'll have a zoo of validation tools. Also it may be still raw and I'm not sure
is it possible to apply it to solve the described problem, but I'd encourage
you to take a look at it.

## Conclusion

ActiveRecord is good for plain and straightforward projects.
In big enterprise applications usually we need more flexebility to meet different
exotic requiments.
We've created [Themis](https://github.com/TMXCredit/themis) to extend ActiveRecord
and solve some of the problems.
Actually I hope that [ROM](http://rom-rb.org/) will be ready soon and we'll
have an ability to select right ORM before diving into development.

Thanks for reading. Hope the article was useful for you and I'm wating for
your feedback!


## Links

* [Themis on Github](https://github.com/TMXCredit/themis) - you'll find here comprehensive documentation in README;
* [Conditional validation](http://guides.rubyonrails.org/active_record_validations.html#conditional-validation) - extraction from Rails Guide;
* [Vanguard on Github](https://github.com/mbj/vanguard) - validator for ROM project;
* [Railscast: #42](http://railscasts.com/episodes/42-with-options) - Ryan Bates describes how `with_options` works.
