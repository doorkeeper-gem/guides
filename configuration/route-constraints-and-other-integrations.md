# Route Constraints and other integrations

You can leverage the `Doorkeeper.authenticate` facade to easily extract a `Doorkeeper::OAuth::Token` based on the current request. You can then ensure that token is still good, find its associated `#resource_owner_id`, etc.

```ruby
module Constraint
  class Authenticated

    def matches?(request)
      token = Doorkeeper.authenticate(request)
      token && token.accessible?
    end
  end
end
```

For more information about integration and other integrations, check out [the related wiki page](https://github.com/doorkeeper-gem/doorkeeper/wiki/ActionController::Metal-with-doorkeeper).

