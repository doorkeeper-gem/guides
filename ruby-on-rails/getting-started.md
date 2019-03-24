# Getting Started

{% hint style="warning" %}
This guide is relevant for **Ruby on Rails** with **ActiveRecord**. It assumes you have an **User** model using [Devise](https://github.com/plataformatec/devise) as the authentication framework.

If you want to see how doorkeeper integrates with an existing application, check out the [doorkeeper-provider-app](https://github.com/doorkeeper-gem/doorkeeper-provider-app/) repository, which is based on this guide.
{% endhint %}

## Installation

The first step is to add Doorkeeper to you project's dependencies:

```text
bundle add doorkeeper
```

After that, you need to generate relevant files with:

```text
bundle exec rails generate doorkeeper:install
```

This will introduce three changes:

1. New **initializer** in `config/initializers/doorkeeper.rb`
2. Add a doorkeeper's **routes** to your `config/routes.rb`
3. **Locale** files in `config/locales/doorkeeper.en.yml`

## Migrations

To generate appropriate tables, run:

```text
$ bundle exec rails generate doorkeeper:migration
    create  db/migrate/20190324080634_create_doorkeeper_tables.rb
```

This migration will create all necessary tables that will hold [oAuth2 Applications](../concepts/application.md), [Access Grants](../concepts/access-grant.md) and [Access Tokens](../concepts/access-token.md). See [the database design](../internals/database-design.md) for more details.

#### Integrating with existing User Model

Before executing the migration, you may want to add foreign keys to the doorkeeper's tables to ensure data integrity. Go to the migration file and uncomment the lines below:

{% code-tabs %}
{% code-tabs-item title="db/migrate/20190324080634\_create\_doorkeeper\_tables.rb" %}
```ruby
# Uncomment below to ensure a valid reference to the resource owner's table
add_foreign_key :oauth_access_grants, :users, column: :resource_owner_id
add_foreign_key :oauth_access_tokens, :users, column: :resource_owner_id
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Now you're ready to run the migrations:

```text
bundle exec rake db:migrate
```

As the next step, you may want to add associations to your model. If you skip this step, you'll encounter `ActiveRecord::InvalidForeignKey`error when you try to destroy the `User` that has associated access grants or access tokens.

{% code-tabs %}
{% code-tabs-item title="app/models/user.rb" %}
```ruby
class User < ApplicationRecord
  has_many :access_grants,
           class_name: 'Doorkeeper::AccessGrant',
           foreign_key: :resource_owner_id,
           dependent: :delete_all # or :destroy if you need callbacks

  has_many :access_tokens,
           class_name: 'Doorkeeper::AccessToken',
           foreign_key: :resource_owner_id,
           dependent: :delete_all # or :destroy if you need callbacks
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}



