# Controllers & Helpers

Doorkeeper ships with seven controllers that handle every OAuth 2.0 endpoint and the built-in admin UI. These controllers are divided into HTML-facing (subclassing `ApplicationController`) and API/metal endpoints (subclassing `ApplicationMetalController`). Controller helpers are mixed into both hierarchies so you can protect your own API resources with a single `before_action`.

## Controllers

All controllers live under `app/controllers/doorkeeper/`. The table below lists each one with its inheritance, public action names (verified from the 5.9.5 source), and purpose.

| Controller | Inherits from | Actions | Purpose |
|---|---|---|---|
| `Doorkeeper::ApplicationController` | `resolve_controller(:base)` | (base class) | Base for HTML/admin controllers. Includes `Helpers::Controller`, sets up `protect_from_forgery` (non-API mode), and registers the `doorkeeper/dashboard` helper. |
| `Doorkeeper::ApplicationMetalController` | `resolve_controller(:base_metal)` | (base class) | Base for API/JSON endpoints. Includes `Helpers::Controller` and a `before_action :enforce_content_type` that rejects non-form-urlencoded POST/PUT/PATCH requests with 415. |
| `Doorkeeper::ApplicationsController` | `ApplicationController` | `index`, `new`, `create`, `show`, `edit`, `update`, `destroy` | Full CRUD admin UI for managing OAuth applications. Uses the `doorkeeper/admin` layout. Protected by `authenticate_admin!`. |
| `Doorkeeper::AuthorizedApplicationsController` | `ApplicationController` | `index`, `destroy` | Lists and revokes applications that the current resource owner has authorized. Protected by `authenticate_resource_owner!`. |
| `Doorkeeper::AuthorizationsController` | `ApplicationController` | `new`, `create`, `destroy` | The `/oauth/authorize` endpoint. `new` renders the grant screen (or auto-authorizes/errors), `create` issues the authorization code, `destroy` denies the request. Protected by `authenticate_resource_owner!`. |
| `Doorkeeper::TokensController` | `ApplicationMetalController` | `create`, `revoke`, `introspect` | API endpoints: `create` (POST /oauth/token — all grant types), `revoke` (POST /oauth/revoke — RFC 7009), `introspect` (POST /oauth/introspect — RFC 7662). Rescues `Errors::DoorkeeperError`. |
| `Doorkeeper::TokenInfoController` | `ApplicationMetalController` | `show` | GET /oauth/token/info. Returns the current token as JSON when `doorkeeper_token` is accessible, or an error response otherwise. |

## Overriding controllers

Use the `controllers` mapping inside `use_doorkeeper` to point a route to your own subclass, then create that subclass in your application.

{% code-tabs %}
{% code-tabs-item title="config/routes.rb" %}
```ruby
Rails.application.routes.draw do
  use_doorkeeper do
    controllers applications: 'oauth/applications'
  end
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

{% code-tabs %}
{% code-tabs-item title="app/controllers/oauth/applications_controller.rb" %}
```ruby
module Oauth
  class ApplicationsController < Doorkeeper::ApplicationsController
    before_action :require_superadmin!

    private

    def require_superadmin!
      head :forbidden unless current_user&.superadmin?
    end
  end
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

The same pattern applies to all controller keys: `:applications`, `:authorized_applications`, `:authorizations`, `:tokens`, and `:token_info`. See [Routes](routes.md) for the full `use_doorkeeper` DSL.

## Controller helpers

Doorkeeper mixes `Doorkeeper::Rails::Helpers` into `ActionController::Base` (via Railtie) and `Doorkeeper::Helpers::Controller` into both base Doorkeeper controllers. Together they provide the following methods.

| Helper | Returns / Does | Available in |
|---|---|---|
| `doorkeeper_token` | Returns the current `Doorkeeper::AccessToken` authenticated from the request (memoized). `nil` if no valid token is present. | All controllers (mixed into `ActionController::Base` by the Railtie). Also defined in `Helpers::Controller` for Doorkeeper's own controllers. |
| `doorkeeper_authorize!(*scopes)` | `before_action` that requires a valid token with **at least one** of the given scopes (logical OR). To require several scopes at the same time, call `doorkeeper_authorize!` once per scope. Renders a 401 JSON response if the token is missing/invalid, or a 403 if the token is valid but has none of the required scopes. | All controllers (via `Doorkeeper::Rails::Helpers`). |
| `valid_doorkeeper_token?` | Returns `true` if the current token is present and satisfies the scopes passed to `doorkeeper_authorize!`. | All controllers. |
| `current_resource_owner` | Returns the resource owner (user) of the current `doorkeeper_token`, evaluated via `Doorkeeper.config.authenticate_resource_owner`. Memoized. Registered as a view helper since 5.9.1. | All Doorkeeper controllers (via `Helpers::Controller`). |
| `doorkeeper_unauthorized_render_options(error:)` | Override hook to customize the 401 response body. Receives the error object. Default is a no-op. | All controllers. |
| `doorkeeper_forbidden_render_options(error:)` | Override hook to customize the 403 response body. Receives the error object. Default is a no-op. | All controllers. |
| `authenticate_resource_owner!` | `before_action` that triggers `current_resource_owner` evaluation. Used by AuthorizationsController and AuthorizedApplicationsController. | Doorkeeper controllers (via `Helpers::Controller`). |
| `authenticate_admin!` | `before_action` evaluated via `Doorkeeper.config.authenticate_admin`. Used by ApplicationsController. | Doorkeeper controllers (via `Helpers::Controller`). |
| `server` | Returns a memoized `Doorkeeper::Server` instance wrapping the current controller. | Doorkeeper controllers. |
| `skip_authorization?` | Evaluates `Doorkeeper.config.skip_authorization` with the resource owner and pre-auth client. | Doorkeeper controllers (used in AuthorizationsController). |
| `enforce_content_type` | `before_action` on `ApplicationMetalController`. Rejects PUT/POST/PATCH requests without `application/x-www-form-urlencoded` content type with 415. | Metal controllers only. |

### Protecting an API controller

The most common pattern is calling `doorkeeper_authorize!` as a `before_action` in your own API controllers:

{% code-tabs %}
{% code-tabs-item title="app/controllers/api/v1/base_controller.rb" %}
```ruby
module Api
  module V1
    class BaseController < ApplicationController
      before_action :doorkeeper_authorize!

      private

      def current_user
        @current_user ||= User.find(doorkeeper_token.resource_owner_id) if doorkeeper_token
      end
    end
  end
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Pass scopes to restrict access further. Multiple scopes are combined with a logical OR, so this accepts a token that has `read` **or** `write`:

```ruby
before_action -> { doorkeeper_authorize! :read, :write }
```

To require both scopes, call the helper once per scope:

```ruby
before_action -> { doorkeeper_authorize! :read }
before_action -> { doorkeeper_authorize! :write }
```

## Customization patterns

### Customizing the token response body

Override `Doorkeeper::TokensController` to modify the JSON returned by `POST /oauth/token`. The `create` action renders `authorize_response.body`, so hook into the response after a successful authorization:

{% code-tabs %}
{% code-tabs-item title="app/controllers/oauth/tokens_controller.rb" %}
```ruby
module Oauth
  class TokensController < Doorkeeper::TokensController
    def create
      super

      if response.status == 200
        body = JSON.parse(response.body)
        body[:user_id] = token.resource_owner_id
        self.response_body = body.to_json
      end
    end
  end
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### Custom authorization flow

Subclass `Doorkeeper::AuthorizationsController` to add pre-authorization checks or a custom grant screen:

{% code-tabs %}
{% code-tabs-item title="app/controllers/oauth/authorizations_controller.rb" %}
```ruby
module Oauth
  class AuthorizationsController < Doorkeeper::AuthorizationsController
    before_action :require_verified_email!

    private

    def require_verified_email!
      return if current_resource_owner&.verified_email?

      render :email_verification_required, status: :forbidden
    end
  end
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### Restricting admin access

Override `Doorkeeper::ApplicationsController` to gate the admin UI behind your own authorization logic:

{% code-tabs %}
{% code-tabs-item title="app/controllers/oauth/applications_controller.rb" %}
```ruby
module Oauth
  class ApplicationsController < Doorkeeper::ApplicationsController
    before_action :require_admin!, except: [:show]

    private

    def require_admin!
      head :forbidden unless current_user&.admin?
    end
  end
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

---

See also: [Routes](routes.md), [Configuration overview](configuration.md), [Testing](../internals/testing.md).
