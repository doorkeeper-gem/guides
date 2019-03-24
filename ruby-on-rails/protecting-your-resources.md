# Securing the API

To protect your controllers \(usual one or `ActionController::API`\) with OAuth, you just need to setup `before_action`s specifying the actions you want to protect. For example:

```ruby
class Api::V1::ProductsController < Api::V1::ApiController
  before_action :doorkeeper_authorize! # Requires access token for all actions
  
  # before_action -> { doorkeeper_authorize! :read, :write }

  # your actions
end
```

You can pass any option `before_action` accepts, such as `if`, `only`, `except`, and others.

#### Authenticated resource owner

If you want to return data based on the current resource owner, in other words, the access token owner, you may want to define a method in your controller that returns the resource owner instance:

```ruby
class Api::V1::CredentialsController < Api::V1::ApiController
  before_action :doorkeeper_authorize!
  respond_to    :json

  # GET /me.json
  def me
    respond_with current_resource_owner
  end

  private

  # Find the user that owns the access token
  def current_resource_owner
    User.find(doorkeeper_token.resource_owner_id) if doorkeeper_token
  end
end
```

In this example, we're returning the credentials \(`me.json`\) of the access token owner.

