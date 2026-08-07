# Models

Starting from Doorkeeper 5.3 you can use your own model classes if you need to extend (or even override)
default Doorkeeper models such as `Application`, `AccessToken` and `AccessGrant`.

By default Doorkeeper ActiveRecord ORM uses it's own classes:

```ruby
# app/initializers/doorkeeper.rb

Doorkeeper.configure do 
  access_token_class "Doorkeeper::AccessToken"
  access_grant_class "Doorkeeper::AccessGrant"
  application_class "Doorkeeper::Application"
end
```

If you're planning to use your own, don't forget inherit them from the base classes (listed above) or
at least to **include** Doorkeeper ORM mixins into your custom models first:

*  `::Doorkeeper::Orm::ActiveRecord::Mixins::AccessToken` - for access token
*  `::Doorkeeper::Orm::ActiveRecord::Mixins::AccessGrant` - for access grant
*  `::Doorkeeper::Orm::ActiveRecord::Mixins::Application` - for application (OAuth2 client)

An example of model customization:

```ruby
Doorkeeper.configure do
  access_token_class "MyAccessToken"
  # ...
end

# app/models/my_access_token.rb
class MyAccessToken < ApplicationRecord
  include ::Doorkeeper::Orm::ActiveRecord::Mixins::AccessToken

  self.table_name = "hey_i_wanna_my_name"
  
  def destroy_me!
    destroy
  end
end
```

### Polymorphic Resource Owner

Enables polymorphic association for the resource owner on Access Tokens and Access Grants. By default, Doorkeeper stores
only the `resource_owner_id` without knowing the owner's model type.

```ruby
Doorkeeper.configure do
  use_polymorphic_resource_owner
end
```

After enabling, generate and run the migration to add the `resource_owner_type` column:

```bash
$ bundle exec rails generate doorkeeper:enable_polymorphic_resource_owner
$ bundle exec rails db:migrate
```

This adds `resource_owner_type` to both `oauth_access_tokens` and `oauth_access_grants` tables. Once enabled, `doorkeeper_token.resource_owner`
returns the proper polymorphic association.

**Important:** If you enable this on an existing project, you must backfill the `resource_owner_type` column for existing records, for example:

```ruby
Doorkeeper::AccessToken.update_all(resource_owner_type: "User")
```

See [Polymorphic Resource Owner](../ruby-on-rails/polymorphic-resource-owner.md) for details.

### Application Owner

Allows each registered OAuth application to have an owner. Disabled by default.

```ruby
Doorkeeper.configure do
  enable_application_owner

  # Optionally require every application to have an owner:
  # enable_application_owner confirmation: true
end
```

Generate the migration to add the owner reference:

```bash
$ bundle exec rails generate doorkeeper:application_owner
$ bundle exec rails db:migrate
```

When enabled, the `Doorkeeper::Application` model includes the `Doorkeeper::Models::Ownership` concern, which adds a polymorphic
`belongs_to :owner` association. With `confirmation: true`, application records are validated to require a non-nil owner.
