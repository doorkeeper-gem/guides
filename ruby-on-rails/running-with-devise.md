# Running Doorkeeper with Devise

Doorkeeper delegates resource-owner authentication to your application via the
`resource_owner_authenticator` block. When you use [Devise](https://github.com/heartcombo/devise)
as your authentication framework, wiring them together takes a few lines of
configuration and one controller override.

## The Authenticator Block

The central piece lives in the Doorkeeper initializer. The block runs in the
context of your Rails application, so `current_user` (from Devise) and `warden`
are available directly:

{% tabs %}
{% tab title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper.configure do
  resource_owner_authenticator do
    # If the user is already signed in, return them.
    # Otherwise, Warden triggers the Devise sign-in flow.
    current_user || warden.authenticate!(scope: :user)
  end
end
```
{% endtab %}
{% endtabs %}

This block is also shown in the [Configuration](configuration.md) guide, which
covers the rest of the Doorkeeper settings.

## Avoiding the Redirect Loop

When an unauthenticated user hits an OAuth endpoint, Doorkeeper calls
`warden.authenticate!` to force sign-in. Devise's `RegistrationsController`
overrides `authenticate_scope!` and calls `authenticate_resource!`
(`authenticate_user!`, `authenticate_admin!`, etc.), which in turn calls
`warden.authenticate!` again -- creating a redirect loop.

To break the loop, override `authenticate_scope!` in your registrations
controller so it looks up the current resource **without** triggering a new
authentication challenge:

{% tabs %}
{% tab title="app/controllers/users/registrations_controller.rb" %}
```ruby
class Users::RegistrationsController < Devise::RegistrationsController
  def authenticate_scope!
    # Instead of calling authenticate_resource! (which triggers
    # warden.authenticate! and causes a redirect loop), just look up
    # the currently signed-in resource.
    self.resource = send(:"current_#{resource_name}")
  end
end
```
{% endtab %}
{% endtabs %}

Make sure this controller is wired into your routes (Devise's default
`registrations` routing will pick it up if you follow the standard
`devise_for :users, controllers: { registrations: 'users/registrations' }`
pattern).

## Exposing the Current Resource Owner

In your API controllers, you need a way to look up the user that owns the
current access token. Use `find_by` rather than `find` to avoid raising an
exception when the token is invalid or the user has been deleted:

{% tabs %}
{% tab title="app/controllers/api/v1/api_controller.rb" %}
```ruby
class Api::V1::ApiController < ApplicationController
  private

  def current_resource_owner
    User.find_by(id: doorkeeper_token&.resource_owner_id)
  end
end
```
{% endtab %}
{% endtabs %}

The `doorkeeper_token` helper is provided by Doorkeeper after a successful
`doorkeeper_authorize!` call. For a full example of protecting controller
actions with `before_action :doorkeeper_authorize!`, see
[Securing the API](protecting-your-resources.md).

## Version Notes

- **Doorkeeper 5.x / 6.x**: The patterns above are current. Older guides may
  reference `doorkeeper_for`, which was removed and should not be used.
- **Password grant**: If you enable the Resource Owner Password Credentials
  grant, Doorkeeper 5.5+ requires explicit client authentication. See
  [Grant Flows](grant-flows.md) for details on configuring grant types.

## See also

- [Configuration](configuration.md) — full Doorkeeper initializer reference
- [Securing the API](protecting-your-resources.md) — `doorkeeper_authorize!` and token-protected controllers
- [Getting Started](getting-started.md) — installation and setup with Devise
- [Grant Flows](grant-flows.md) — available OAuth 2.0 grant types
