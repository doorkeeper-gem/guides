# Customizing Views

Doorkeeper ships with a full set of default views for the authorization grant screen, the OAuth application management UI, and the authorized applications list.
You can override any of them by generating the view templates into your application and customizing them to match your design.

## Generating views

Run the built-in generator to copy all Doorkeeper view templates into your application:

```bash
rails generate doorkeeper:views
```

This places every template under `app/views/doorkeeper/`. Doorkeeper looks for views in this directory first; if a template
is not found there, it falls back to the default one shipped with the gem.

After generating, you can edit any file in `app/views/doorkeeper/` and your changes will take effect immediately.

## Custom layouts

By default, Doorkeeper controllers render within their own layout. You can point each controller to your application's
layout by adding a `config.to_prepare` block in `config/application.rb`:

{% tabs %}
{% tab title="config/application.rb" %}
```ruby
config.to_prepare do
  # Only Applications list
  Doorkeeper::ApplicationsController.layout "my_layout"

  # Only Authorization endpoint
  Doorkeeper::AuthorizationsController.layout "my_layout"

  # Only Authorized Applications
  Doorkeeper::AuthorizedApplicationsController.layout "my_layout"
end
```
{% endtab %}
{% endtabs %}

To apply the same layout to all Doorkeeper controllers at once, set it on the base class:

{% tabs %}
{% tab title="config/application.rb" %}
```ruby
config.to_prepare do
  Doorkeeper::ApplicationController.layout "my_layout"
end
```
{% endtab %}
{% endtabs %}

You can also set the layout in an initializer (`config/initializers/doorkeeper.rb`), though `config.to_prepare` in `config/application.rb`
is preferred because it runs after all initializers have loaded, ensuring your application's layout helpers are available.

## The `main_app.` prefix

Doorkeeper is an **isolated engine**. Inside its views, the routing context is Doorkeeper's own engine, not your main
Rails application. This means every route helper and controller helper that belongs to your application must be prefixed with `main_app.`:

{% tabs %}
{% tab title="app/views/doorkeeper/authorizations/new.html.erb" %}
```erb
<% if main_app.user_signed_in? %>
  <%= link_to "Sign Out", main_app.destroy_user_session_path, method: :delete %>
<% else %>
  <%= link_to "Sign In", main_app.new_user_session_path %>
<% end %>
```
{% endtab %}
{% endtabs %}

This is the most common point of confusion when customizing Doorkeeper views. Forgetting `main_app.` will raise `NoMethodError` or produce broken links because the helper or route is looked up in the Doorkeeper engine namespace instead of your application's.

## Making application helpers available

If you have helper methods in `ApplicationHelper` (or any other helper module) that you want to use directly inside Doorkeeper views without the `main_app.` prefix, include them in the Doorkeeper base controller:

{% tabs %}
{% tab title="config/application.rb" %}
```ruby
config.to_prepare do
  # Include only the ApplicationHelper module
  Doorkeeper::ApplicationController.helper ApplicationHelper

  # Include all helpers from your application
  Doorkeeper::ApplicationController.helper YourApp::Application.helpers
end
```
{% endtab %}
{% endtabs %}

Once included, your helper methods become available in Doorkeeper views just as they are in your own views.
This is optional — you can always access application helpers through `main_app.` without any configuration change.

---

See also: [Controllers & Helpers](controllers-and-helpers.md), [Configuration](configuration.md), [Getting Started](getting-started.md).
