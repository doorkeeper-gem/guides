# Scopes

#### Access Token Scopes

You can also require the access token to have specific scopes in certain actions:

First configure the scopes in `initializers/doorkeeper.rb`

```ruby
Doorkeeper.configure do
  default_scopes :public # if no scope was requested, this will be the default
  optional_scopes :admin, :write
end
```

And in your controllers:

```ruby
class Api::V1::ProductsController < Api::V1::ApiController
  before_action -> { doorkeeper_authorize! :public }, only: :index
  before_action only: [:create, :update, :destroy] do
    doorkeeper_authorize! :admin, :write
  end
end
```

Please note that there is a logical OR between multiple required scopes. In the above example, `doorkeeper_authorize! :admin, :write` means that the access token is required to have either `:admin` scope or `:write` scope, but does not need to have both of them.

If you want to require the access token to have multiple scopes at the same time, use multiple `doorkeeper_authorize!`, for example:

```ruby
class Api::V1::ProductsController < Api::V1::ApiController
  before_action -> { doorkeeper_authorize! :public }, only: :index
  before_action only: [:create, :update, :destroy] do
    doorkeeper_authorize! :admin
    doorkeeper_authorize! :write
  end
end
```

In the above example, a client can call `:create` action only if its access token has both `:admin` and `:write` scopes.

### Dynamic Scopes

When your scope list is not known ahead of time (for example, `user:1`, `user:2`, …), you can enable dynamic scopes.
Dynamic scope patterns use a delimiter (default `:`) to separate the scope name from a matching value, and they bypass the
statically configured `default_scopes` / `optional_scopes` lists.

```ruby
Doorkeeper.configure do
  enable_dynamic_scopes

  # Default delimiter is ":", producing patterns like "user:*"
  # Customize it with:
  # enable_dynamic_scopes delimiter: "."
end
```

With dynamic scopes enabled, clients can request scopes like `user:123` or `repo:42` even though those are not listed in
`default_scopes` or `optional_scopes`. Doorkeeper will match them against patterns such as `user:*`.

### Scopes by Grant Type

You can restrict which scopes are available per OAuth 2.0 grant flow. By default all configured scopes are available for every grant type.

```ruby
Doorkeeper.configure do
  scopes_by_grant_type(
    authorization_code: [:read, :write],
    client_credentials: [:public],
    password: [:write],
  )
end
```

A client using the `client_credentials` grant will only be able to request the `:public` scope, even though other scopes exist in the configuration.

### Enforcing Configured Scopes

By default, applications can be created or updated with arbitrary scopes. Enable `enforce_configured_scopes` to forbid scopes
that are not declared in `default_scopes` or `optional_scopes`.

```ruby
Doorkeeper.configure do
  enforce_configured_scopes
end
```

When enabled, any attempt to set scopes outside the configured list on an application will be rejected.

