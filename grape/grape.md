# Grape

Starting from version 2.2 Doorkeeper provides helpers for the [Grape framework](https://github.com/ruby-grape/grape) &gt;= 0.10. One of them is `doorkeeper_authorize!` that can be used in a similar way as an example above to protect your API with OAuth. Note that you have to use `require 'doorkeeper/grape/helpers'` and `helpers Doorkeeper::Grape::Helpers` in your Grape API class.

For more information about integration with Grape see the [Wiki](https://github.com/doorkeeper-gem/doorkeeper/wiki/Grape-Integration).

```ruby
require 'doorkeeper/grape/helpers'

module API
  module V1
    class Users < Grape::API
      helpers Doorkeeper::Grape::Helpers

      before do
        doorkeeper_authorize!
      end

      # route_setting :scopes, ['user:email'] - for old versions of Grape
      get :emails, scopes: [:user, :write] do
        [{'email' => current_user.email}]
      end

      # ...
    end
  end
end
```

