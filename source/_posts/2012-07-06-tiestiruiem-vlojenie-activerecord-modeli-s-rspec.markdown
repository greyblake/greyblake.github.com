---
layout: post
title: "Тестируем вложенные ActiveRecord-модели с RSpec"
date: 2012-07-06 22:19
comments: true
categories: Ruby, RSpec, test, unit, ActiveRecord, Ruby on Rails, тест, модель
---

Иногда бывает так, что вам нужно построить большой граф вложенных объектов,
и конечно же протестировать, что ваш "builder" работает так, как нужно. На самом деле
задача элементарная, но я всё же попробую поискать наиболее элегантный путь её решения.

Расмотрим следующий пример, когда у нас есть три небольшие модели: User, Account, Preference.
(На практики обычно моделей намного больше с большим количеством свойств).

```ruby user.rb
class User < ActiveRecord::Base
  attr_accessible :last_name, :first_name

  has_one :account
end
```

```ruby account.rb
class Account < ActiveRecord::Base
  attr_accessible :email, :last_visit_date

  belongs_to :user
  has_one :preference
end
```

```ruby preference.rb
class Preference < ActiveRecord::Base
  attr_accessible :language, :weapon

  belongs_to :user
end
```

Предположим у нас есть некий builder, который строит объект User и все завимимые
модели(Account и Preference). Всё что мы хотим сделать - это протестировать,
что граф объектов построен правильно.

Вот пример реализации стандартного rspec-теста, который первым приходит на ум:

```ruby user_builder_spec.rb
describe UserBuilder do
  describe '#build' do
    describe 'user' do
      let(:user) { described_class.new(some_attrs).build }
      subject { user }

      its(:first_name) { should == "Rodion"      }
      its(:last_name)  { should == "Raskolnikov" }

      describe 'account' do
        subject { user.account }

        its(:email)           { should == "rodion@mail.ru"       }
        its(:last_visit_date) { should == Date.new(1866, 11, 27) }

        describe 'preference' do
          subject { user.account.preference }

          its(:language) { should == "Russian" }
          its(:weapon)   { should == "ax"      }
        end
      end
    end
  end
end
```

Всё читабельно, красиво и просто. Но проблема в том, что получилось шесть тестов(по тесту на
каждый атрибут), которые тестируют лишь одно действие - постороение объекта. Когда операция построения
занимает немало времени, а количество атрибутов на порядок больше, такой подход
становится далеко не самым лучшим, поскольку возрастает время выполнения.
Если объект не сохраняется в базе то вполне разумным будет использовать `before :all`. Но если
вам приходится сохранять объект и вы используете опцию `config.use_transactional_fixtures = true`,
потому что не хотите "гадить" в базу, то этот вариант не подойдёт.

Можно пойти по пути классического unit-тестирования и сделать тест подобным этому:

```ruby user_builder_spec.rb
describe UserBuilder do
  describe '#build' do
    let(:user) { described_class.new(some_attrs).build }

    it 'should correctly build a user' do
      user.first_name.should == "Rodion"
      user.last_name.should  == "Raskolnikov"
      user.account.email.should           == "rodion@mail.ru"
      user.account.last_visit_date.should == Date.new(1866, 11, 27)
      user.account.preference.language.should == "Russian"
      user.account.preference.weapon.should   == "ax"
    end
  end
end
```

Получилось даже лаконичнее, но менее читабельно(уж когда будет больше вложенность объектов,
будет точно менее читабельно). Так же раздрожает длинные цепочки одинаковых методов.

Помедитировав над проблемой, я нашёл компроммиссное решение на основе метода `instance_eval`.

```ruby user_builder_spec.rb
describe UserBuilder do
  describe '#build' do
    let(:user) { described_class.new(some_attrs).build }

    it 'should correctly build a user' do
      user.instance_eval do
        first_name.should == "Rodion"
        last_name.should  == "Raskolnikov"

        account.instance_eval do
          email.should           == "rodion@mail.ru"
          last_visit_date.should == Date.new(1866, 11, 27)

          preference.instance_eval do
            language.should == "Russian"
            weapon.should   == "ax"
          end
        end
      end
    end

  end
end
```

Это по прежднему один тест, но мы избавились от длинных цепочек методов.
Нужно заметить, что у такого подхода есть тоже свои недостатки: используя `instance_eval`
мы покидаем контекст теста, и переходим прямо в контекст объекта, в котором не существует методов подобных
`be_valid`, `be_instance_of`, etc.

Надеюсь, эта идея будет вам полезной. Буду рад узнать чужое мнение.
