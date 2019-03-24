# Scopes

For this guide let's create two scopes: `read` and `write`. Applications authorized with the `read` scope will be able to read public data from our API. The `write` scope will let applications change data in our API.

Go to doorkeeper's initializer and add:

{% code-tabs %}
{% code-tabs-item title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper.configure do
  default_scopes :read
  optional_scopes :write
    
  enforce_configured_scopes
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

The last line with `enforce_configured_scopes` ensures applications to be able to ask only for configured scopes defined in `default_scopes` and `optional_scopes`.

