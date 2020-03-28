# Other Configurations

## Custom Access Token Generator

By default a 128 bit access token will be generated. If you require a custom token, such as [JWT](http://jwt.io/),
specify an object that responds to `.generate(options = {})` and returns a string to be used as the token.

```ruby
Doorkeeper.configure do
  access_token_generator "Doorkeeper::JWT"
end
```

JWT token support is available with [Doorkeeper-JWT](https://github.com/chriswarren/doorkeeper-jwt).

## Custom controllers

By default Doorkeeper's main controller `Doorkeeper::ApplicationController` inherits from `ActionController::Base`.
You may want to use your own controller to inherit from, to keep Doorkeeper controllers in the same context than
the rest your app:

```ruby
Doorkeeper.configure do
  base_controller 'ApplicationController'
end
```

Also you could override `Doorkeeper::ApplicationMetalController` that is `ActionController::API` by default.

```ruby
Doorkeeper.configure do
  base_metal_controller 'ApplicationController'
end
```

## Customizing errors

If you don't want to use default Doorkeeper error responses you can raise and rescue it's exceptions.
All you need is to set configuration option `handle_auth_errors` to `:raise`. In this case Doorkeeper
will raise `Doorkeeper::Errors::TokenForbidden`, `Doorkeeper::Errors::TokenExpired`,
`Doorkeeper::Errors::TokenRevoked` or other exceptions that you need to care about.

## Other customizations

* [Associate users to OAuth applications \(ownership\)](https://github.com/doorkeeper-gem/doorkeeper/wiki/Associate-users-to-OAuth-applications-%28ownership%29)
* [CORS - Cross Origin Resource Sharing](https://github.com/doorkeeper-gem/doorkeeper/wiki/%5BCORS%5D-Cross-Origin-Resource-Sharing)
* see more on [Wiki page](https://github.com/doorkeeper-gem/doorkeeper/wiki)
