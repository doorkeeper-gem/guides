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

