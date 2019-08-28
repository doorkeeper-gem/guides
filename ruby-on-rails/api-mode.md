# API Mode

By default Doorkeeper uses full Rails stack to provide all the OAuth 2 functionality with additional features like administration area for managing applications. By the way, starting from Doorkeeper 5 you can use API mode for your [API only Rails 5 applications](http://edgeguides.rubyonrails.org/api_app.html). All you need is just to configure the gem to work in desired mode:

{% code-tabs %}
{% code-tabs-item title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper.configure do
  api_only
  base_controller 'ActionController::API'
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Keep in mind, that in this mode you will not be able to access `Applications` or `Authorized Applications` controllers because they will be skipped. CSRF protections \(which are otherwise enabled\) will be skipped, and all the redirects will be returned as JSON response with corresponding locations.

In order to add Applications for authorizing, you will need to create them via console:

```ruby
Doorkeeper::Application.create(name: "MyApp", redirect_uri: "urn:ietf:wg:oauth:2.0:oob", scopes: ["read", "write"])
```
