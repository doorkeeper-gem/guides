# Getting Started

{% hint style="warning" %}
This guide is relevant for Ruby on Rails with Active Record. It assumes you have an `User` model and a way of logging them in.
{% endhint %}

## Installation

Installation depends on the framework you're using. The first step is to add Doorkeeper to you project's dependencies:

```text
bundle add doorkeeper
```

### Generators

After that, you need to generate relevant files with:

```text
bundle exec rails generate doorkeeper:install
```

This will generate 3 files

1. Initializer for doorkeeper in `config/initializers/doorkeeper.rb`
2. Locale files in `config/locales/doorkeeper.en.yml`
3. Add a doorkeeper's routes to your `config/routes.rb`

### Migrations

To generate appropriate tables, run:

```text
bundle exec rails generate doorkeeper:migration
```

This will generate all necessary tables that will hold [applications](../concepts/application.md) and tokens.

You may want to add foreign keys to your migration. For example, if you plan on using `User` as the resource owner, add the following line to the migration file for each table that includes a `resource_owner_id` column:

```text
add_foreign_key :table_name, :users, column: :resource_owner_id
```

If you want to enable [PKCE flow](https://tools.ietf.org/html/rfc7636) for mobile apps, you need to generate another migration:

```text
bundle exec rails generate doorkeeper:pkce
```

Then run migrations:

```text
bundle exec rake db:migrate
```

Ensure to use non-confidential apps for pkce. PKCE is created, because you cannot trust its apps' secret. So whatever app needs pkce: it means, it cannot be a confidential app by design.

Remember to add associations to your model so the related records are deleted. If you don't do this an `ActiveRecord::InvalidForeignKey`-error will be raised when you try to destroy a model with related access grants or access tokens.

{% code-tabs %}
{% code-tabs-item title="app/models/user.rb" %}
```ruby
class User < ApplicationRecord
  has_many :access_grants, class_name: "Doorkeeper::AccessGrant",
                           foreign_key: :resource_owner_id,
                           dependent: :delete_all # or :destroy if you need callbacks

  has_many :access_tokens, class_name: "Doorkeeper::AccessToken",
                           foreign_key: :resource_owner_id,
                           dependent: :delete_all # or :destroy if you need callbacks
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### Server Configuration

Doorkeeper installs itself in `config/routes.rb` and defines routes for managing applications, authorizing users and so on. More details in [Routes](routes.md) page.

Before you're able to use Doorkeeper, you need to configure who can manage [applications](../concepts/application.md) and how [resource owners](../concepts/resource-owner.md) \(users\) can be authenticated.

### Resource Owner Authentication

You need to configure Doorkeeper in order to provide `resource_owner` model and authentication block in `config/initializers/doorkeeper.rb`:

{% code-tabs %}
{% code-tabs-item title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper.configure do
  resource_owner_authenticator do
    User.find_by_id(session[:user_id]) || redirect_to(login_url)
  end
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

This code is run in the context of your application so you have access to your models, session or routes helpers. However, since this code is not run in the context of your application's `ApplicationController` it doesn't have access to the methods defined over there.

You may want to check other ways of authentication [here](https://github.com/doorkeeper-gem/doorkeeper/wiki/Authenticating-using-Clearance-or-DIY).

### Application Management Authentication

By default, the applications list \(`/oauth/applications`\) is publicly available \(before 5.0 release\). Starting from Doorkeeper 5.0 it returns 403 Forbidden if `admin_authenticator` option is not configured by developers.

To change the protection rules of this endpoint you should uncomment these lines:

{% code-tabs %}
{% code-tabs-item title="config/initializers/doorkeeper.rb" %}
```ruby
# config/initializers/doorkeeper.rb
Doorkeeper.configure do
  admin_authenticator do |routes|
    Admin.find_by(id: session[:admin_id]) || redirect_to(routes.new_admin_session_url)
  end
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

The logic is the same as the `resource_owner_authenticator` block. **Note:** since the application list is just a scaffold, it's recommended to either customize the controller used by the list or skip the controller all together. For more information see the page [in the wiki](https://github.com/doorkeeper-gem/doorkeeper/wiki/Customizing-routes).

By default, everybody can create application with any scopes. However, you can enforce users to create applications only with configured scopes \(`default_scopes` and `optional_scopes` from the Doorkeeper initializer\):

{% code-tabs %}
{% code-tabs-item title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper.configure do
  default_scopes :read, :write
  optional_scopes :create, :update

  enforce_configured_scopes
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}



