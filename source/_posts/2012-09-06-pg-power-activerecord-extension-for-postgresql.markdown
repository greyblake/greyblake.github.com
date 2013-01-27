---
layout: post
title: "pg_power - ActiveRecord extension for PostgreSQL"
date: 2012-09-06 23:24
comments: true
categories: pg pogstgresql unix linux ruby pg_power rails
---

I am happy to announce that  [TMXCredit](http://tmxcredit.com/) released
[pg_power](https://github.com/TMXCredit/pg_power) gem - an ActiveRecord extension which
allows to use number of PostgreSQL features with Rails.


## What you can do with pg_power?

* Use PostgresSQL schemas in your Rails project.
* Add comments to PostgreSQL database with Rails migrations.
* Use foreign keys (we imported foreigner functionality and made it schema aware).
* Use partial indexes.
* Add indexes concurrently.

You'll find enough documentation in [README](https://github.com/TMXCredit/pg_power/blob/master/README.markdown)
file.


## Quick usage example

Assume you want to create tables `countries` and `languages` in `demography` schema.

At first we need to create `demography` schema:

```ruby db/migrate/create_demography_schema.rb
class CreateDemographySchema < < ActiveRecord::Migration
  def change
    create_schema 'demography'
  end
end
```

Now let's create tables:

```ruby db/migrate/create_demography_languages.rb
class CreateDemographyLanguages < ActiveRecord::Migration
  def change
    # Create table `languages` in schema `demography`
    create_table "languages", :schema => "demography" do |t|
      t.string :name
      t.string :code, :limit => 2
    end

    # Add PostgreSQL comments
    set_table_comment "demography.languages", "List of languages"
    set_column_comments "demography.languages",
        :name => "Full name of language in English",
        :code => "ISO 639-1 code"
  end
end
```

```ruby db/migrate/create_demography_countries.rb
class CreateDemographyContries < ActiveRecord::Migration
  def change
    # Create table `countries` in schema `demography`
    create_table "countries", :schema => "demography" do |t|
      t.string :name

      # In real life you likely would have many-to-many associaton
      t.integer :language_id
    end

    # Add PostgreSQL comments
    set_table_comment "demography.countries", "List of world countries"
    set_column_comments "demography.languages",
        :name        => "Full name of country in English",
        :language_id => "Most popular language in the country"

    # Add foreign key and create index on demography.countries.language_id
    add_foreign_key("demography.countries", "demography.languages")
  end
end
```

Great! Now we need to set table names in models to make ActiveRecord know that
these tables are located in `demography` schema.

```ruby app/models/language.rb
class Language < ActiveRecord::Base
  set_table_name "demography.languages"
end
```

It will work. But I would recommend you to create module `Demography` which would represent
`demography` schema and move those models to it. One more benefit is that you can define
schema prefix in module and models will use it build table name automatically.

```ruby app/models/demography.rb
module Demography
  def self.table_name_prefix
    'demography.'
  end
end
```

```ruby app/models/demography/language.rb
module Demography::Language
  # No need to use set_table_name anymore
end
```


I hope you will enjoy [pg_power](https://github.com/TMXCredit/pg_power). Let us know what you think!

Thanks. Sergey Potapov.
