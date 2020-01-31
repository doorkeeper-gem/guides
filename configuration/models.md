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
