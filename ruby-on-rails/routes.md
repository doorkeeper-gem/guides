# Routes

The installation script will also automatically add the Doorkeeper routes into your app:

{% tabs %}
{% tab title="config/routes.rb" %}
```ruby
Rails.application.routes.draw do
  use_doorkeeper
  # your routes below
end
```
{% endtab %}
{% endtabs %}

This will mount following routes:

```text
   native_oauth_authorization GET    /oauth/authorize/native(.:format)             doorkeeper/authorizations#show
          oauth_authorization GET    /oauth/authorize(.:format)                    doorkeeper/authorizations#new
                              DELETE /oauth/authorize(.:format)                    doorkeeper/authorizations#destroy
                              POST   /oauth/authorize(.:format)                    doorkeeper/authorizations#create
                  oauth_token POST   /oauth/token(.:format)                        doorkeeper/tokens#create
                 oauth_revoke POST   /oauth/revoke(.:format)                       doorkeeper/tokens#revoke
             oauth_introspect POST   /oauth/introspect(.:format)                   doorkeeper/tokens#introspect
           oauth_applications GET    /oauth/applications(.:format)                 doorkeeper/applications#index
                              POST   /oauth/applications(.:format)                 doorkeeper/applications#create
        new_oauth_application GET    /oauth/applications/new(.:format)             doorkeeper/applications#new
       edit_oauth_application GET    /oauth/applications/:id/edit(.:format)        doorkeeper/applications#edit
            oauth_application GET    /oauth/applications/:id(.:format)             doorkeeper/applications#show
                              PATCH  /oauth/applications/:id(.:format)             doorkeeper/applications#update
                              PUT    /oauth/applications/:id(.:format)             doorkeeper/applications#update
                              DELETE /oauth/applications/:id(.:format)             doorkeeper/applications#destroy
oauth_authorized_applications GET    /oauth/authorized_applications(.:format)      doorkeeper/authorized_applications#index
 oauth_authorized_application DELETE /oauth/authorized_applications/:id(.:format)  doorkeeper/authorized_applications#destroy
             oauth_token_info GET    /oauth/token/info(.:format)                   doorkeeper/token_info#show

```

## Customizing routes

The `use_doorkeeper` method accepts options and an optional block for customizing the generated
routes. The following options are available:

| Option | Description | Default |
|---|---|---|
| `scope` | URL path prefix for all OAuth routes. | `"oauth"` |

Inside the `use_doorkeeper` block, you can further customize which controllers are mounted
and how they are named:

| Method | Description | Example |
|---|---|---|
| `skip_controllers` | Exclude one or more controllers from route generation. Valid values: `:applications`, `:authorized_applications`, `:authorizations`, `:tokens`, `:token_info`. | `skip_controllers :applications, :token_info` |
| `controllers` | Override a controller class for a given route key. Valid keys: `:applications`, `:authorized_applications`, `:authorizations`, `:tokens`, `:token_info`. | `controllers applications: 'custom_applications'` |
| `as` | Override the named-route helper prefix for a route group. Valid keys: `:authorizations`, `:tokens`, `:token_info`. | `as tokens: :api_token` |

### Examples

Customize the path prefix and skip the applications management UI:

{% tabs %}
{% tab title="config/routes.rb" %}
```ruby
Rails.application.routes.draw do
  use_doorkeeper scope: 'auth' do
    skip_controllers :applications, :authorized_applications
  end
end
```
{% endtab %}
{% endtabs %}

Use a custom controller for managing OAuth applications:

{% tabs %}
{% tab title="config/routes.rb" %}
```ruby
Rails.application.routes.draw do
  use_doorkeeper do
    controllers applications: 'admin/oauth_applications'
  end
end
```
{% endtab %}
{% endtabs %}

For additional customizations, see [this page on the wiki](https://github.com/doorkeeper-gem/doorkeeper/wiki/Customizing-routes).



