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

## Configuration Reference

### Token Expiry and Reuse

| Option | Type | Default | Description |
|---|---|---|---|
| `access_token_expires_in` | `Integer` | `7200` (2 hours) | TTL for access tokens in seconds. Set to `nil` to disable expiry (long explicit TTL preferred over `nil`). |
| `custom_access_token_expires_in` | `Proc` | `->(_context) { nil }` | Per-context TTL override. Receives the grant context; return `nil` to fall back to `access_token_expires_in`, or `Float::INFINITY` for a non-expiring token. |
| `authorization_code_expires_in` | `Integer` | `600` (10 minutes) | TTL for authorization codes in seconds. |
| `reuse_access_token` | flag | off | Reuse an existing valid access token for the same resource owner, application, and scopes instead of issuing a new one. Incompatible with `hash_token_secrets`. |
| `token_reuse_limit` | `Integer` | `100` | Percentage threshold (0–100) capping token reuse when `reuse_access_token` is enabled. |
| `token_lookup_batch_size` | `Integer` | `10_000` | Batch size for database queries when searching for a reusable token. |
| `revoke_previous_client_credentials_token` | flag | off | Revoke the previous client-credentials access token for the same client when a new one is issued. |
| `revoke_previous_authorization_code_token` | flag | off | Revoke the previous authorization-code access token for the same client when a new one is issued. |
| `custom_access_token_attributes` | `Array` | `[]` | List of extra attribute names (e.g. `:tenant_id`) stored on access grants and tokens. Declare these before issuing tokens. |
| `default_generator_method` | `Symbol` | `:urlsafe_base64` | The `SecureRandom` method used by the default token generator. |

### Grant Flows

| Option | Type | Default | Description |
|---|---|---|---|
| `grant_flows` | `Array<String>` | `%w[authorization_code client_credentials]` | Enabled OAuth 2.0 grant flows. Supported values: `authorization_code`, `client_credentials`, `password`, `implicit`, `refresh_token`. |
| `use_refresh_token` | flag / `Proc` | off | Issue refresh tokens alongside access tokens. Pass a block to conditionally enable per-grant: `use_refresh_token { \|grant\| grant.scopes.include?('offline_access') }`. |
| `allow_grant_flow_for_client` | `Proc` | `->(_grant_flow, _client) { true }` | Per-client gate on grant flows. Return `false` to respond with `unauthorized_client`. |
| `skip_client_authentication_for_password_grant` | `Boolean` | `false` | Allow password grants without client authentication. Discouraged by the OAuth 2.0 spec. |
| `authorize_resource_owner_for_client` | `Proc` | `->(_client, _resource_owner) { true }` | Per-client resource-owner authorization check. Return `false` to block a resource owner from authorizing a specific application. |

### Redirect URIs

| Option | Type | Default | Description |
|---|---|---|---|
| `native_redirect_uri` | `String` | `"urn:ietf:wg:oauth:2.0:oob"` | **Deprecated.** URI used for native applications. |
| `force_ssl_in_redirect_uri` | `Boolean` / `Proc` | `!Rails.env.development?` | Force HTTPS in non-native redirect URIs. Pass a callable for fine-grained control (e.g. allow `localhost` in development). |
| `forbid_redirect_uri` | `Proc` | `->(_uri) { false }` | Block specific redirect URIs by returning `true`. Commonly used to forbid `javascript:` URIs. |
| `allow_blank_redirect_uri` | `Proc` | dynamic — allows blank when grant flows exclude `authorization_code` and `implicit` | Permit applications to register without a redirect URI. Useful for client-credentials-only clients. Requires dropping the database `NOT NULL` constraint on the redirect URI column. |
| `use_url_path_for_native_authorization` | flag | off | Change the native authorization response route from `oauth/authorize/native?code=<code>` to `oauth/authorize/<code>`. |

### Application Owner

| Option | Type | Default | Description |
|---|---|---|---|
| `enable_application_owner` | flag / `Hash` | off | Assign an owner to each registered OAuth application. Requires the `application_owner` generator and migration. |
| `confirm_application_owner` | flag | off | Enforce that an application must have an owner (implied when calling `enable_application_owner confirmation: true`). |

### Token / Client Authentication Methods

| Option | Type | Default | Description |
|---|---|---|---|
| `access_token_methods` | `Array<Symbol>` | `%i[from_bearer_authorization from_access_token_param from_bearer_param]` | How the access token is extracted from the request. Order matters: first match wins. |
| `client_credentials_methods` | `Array<Symbol>` | `%i[from_basic from_params]` | **Deprecated in 6.0** — use `client_authentication` instead. How client credentials (ID and secret) are extracted from the request. Order matters: first match wins. |

### Error Handling and Content

| Option | Type | Default | Description |
|---|---|---|---|
| `realm` | `String` | `"Doorkeeper"` | Value for the `WWW-Authenticate` realm header in error responses. |
| `handle_auth_errors` | `Symbol` | `:render` | Set to `:raise` to raise exceptions (`Doorkeeper::Errors::InvalidToken`, etc.) instead of rendering JSON error responses. Set to `:redirect` for RFC 6749 section 4.1.2.1 redirect behaviour. |
| `enforce_content_type` | flag | off | Require token requests to use `application/x-www-form-urlencoded` as mandated by the OAuth 2.0 spec. |

### Token Introspection

| Option | Type | Default | Description |
|---|---|---|---|
| `allow_token_introspection` | `Boolean` / `Proc` | dynamic — permits introspection when the requesting token belongs to the same application, or the authorized client matches the token's application, or the token is public | Control who may introspect a token. Return `false` to reject introspection. Set to `false` to disable completely. See [Token Introspection](token-introspection.md). |
| `custom_introspection_response` | `Proc` | `->(_token, _context) { {} }` | Add custom fields (e.g. `sub`, `aud`) to the [RFC 7662](https://tools.ietf.org/html/rfc7662) introspection response. Receives the token and the request context. |

### Multi-Database Support

| Option | Type | Default | Description |
|---|---|---|---|
| `enable_multiple_database_roles` | flag | off | Wrap database writes to the primary when Rails automatic role switching is enabled. Prevents `ActiveRecord::ReadOnlyError` with read replicas (Rails 6.1+). |

### Lifecycle Hooks

| Option | Type | Default | Description |
|---|---|---|---|
| `before_successful_authorization` | `Proc` | `->(_controller, _context = nil) {}` | Called before completing the authorization flow. Useful for single sign-out or audit logging. |
| `after_successful_authorization` | `Proc` | `->(_controller, _context = nil) {}` | Called after a successful authorization with access to the issued token via the context. |
| `before_successful_strategy_response` | `Proc` | `->(_request) {}` | Called before a token strategy renders its success response (e.g. before the token response body is built). |
| `after_successful_strategy_response` | `Proc` | `->(_request, _response) {}` | Called after a token strategy renders its success response. Receives the request and the response body. |

---

## Token Expiry and Reuse

Access tokens expire after 2 hours by default (`access_token_expires_in`). Override per-grant with
`custom_access_token_expires_in`, which receives the grant context and may return a custom TTL,
`nil` (fall back to the global default), or `Float::INFINITY` for a non-expiring token.

```ruby
Doorkeeper.configure do
  access_token_expires_in 30.minutes

  custom_access_token_expires_in do |context|
    context.grant_type == Doorkeeper::OAuth::CLIENT_CREDENTIALS ? 1.hour : nil
  end
end
```

Authorization codes are short-lived (10 minutes by default) and are one-time-use only.
See [Models](models.md) for the underlying `Doorkeeper::AccessToken` and `Doorkeeper::AccessGrant` models.

Token reuse (`reuse_access_token`) avoids issuing a new token when an existing valid token
matches the same resource owner, application, and scopes. The `token_reuse_limit` (0–100)
caps how often a token may be reused as a percentage. When searching for a reusable token,
results are batched via `token_lookup_batch_size`.

```ruby
Doorkeeper.configure do
  reuse_access_token
  token_reuse_limit 90
  token_lookup_batch_size 5_000
end
```

If you want only one active token per client, enable one or both of:

```ruby
Doorkeeper.configure do
  revoke_previous_client_credentials_token
  revoke_previous_authorization_code_token
end
```

These revoke the previous token of the corresponding grant type when a new one is issued.
Use `custom_access_token_attributes` to store extra data on tokens (e.g. `:tenant_id`):

```ruby
Doorkeeper.configure do
  custom_access_token_attributes [:tenant_id]
end
```

The default token generator calls `SecureRandom.urlsafe_base64`. Override with `default_generator_method`
to use a different method from `SecureRandom`, or set `access_token_generator` for a fully custom generator class.

## Grant Flows

Enable desired grant flows with `grant_flows`:

```ruby
Doorkeeper.configure do
  grant_flows %w[authorization_code client_credentials password refresh_token]
end
```

Available flows: `authorization_code`, `client_credentials`, `password` (resource owner password credentials),
`implicit`, and `refresh_token`. When the `refresh_token` flow is listed, you must also enable
`use_refresh_token`:

```ruby
Doorkeeper.configure do
  use_refresh_token

  # Conditionally:
  use_refresh_token do |grant|
    grant.scopes.include?('offline_access')
  end
end
```

Gate grant flows per client with `allow_grant_flow_for_client`:

```ruby
Doorkeeper.configure do
  allow_grant_flow_for_client do |grant_flow, client|
    client.confidential? || grant_flow != 'client_credentials'
  end
end
```

The `authorize_resource_owner_for_client` hook controls which resource owners may authorize which
applications:

```ruby
Doorkeeper.configure do
  authorize_resource_owner_for_client do |client, resource_owner|
    resource_owner.admin? || !client.blocked?
  end
end
```

The discouraged `skip_client_authentication_for_password_grant` option is available for legacy
integrations but violates the OAuth 2.0 specification.

For Proof Key for Code Exchange, see [PKCE Flow](../ruby-on-rails/pkce-flow.md).

## Redirect URIs

By default Doorkeeper enforces HTTPS for non-native redirect URIs outside the development
environment (`force_ssl_in_redirect_uri`). Override with a callable for exceptions:

```ruby
Doorkeeper.configure do
  force_ssl_in_redirect_uri do |uri|
    uri.host != 'localhost'
  end
end
```

Block dangerous redirect URIs with `forbid_redirect_uri`:

```ruby
Doorkeeper.configure do
  forbid_redirect_uri do |uri|
    uri.scheme.to_s.downcase == 'javascript'
  end
end
```

If your clients use only URI-less grant flows (e.g. client credentials), you can allow blank
redirect URIs by enabling `allow_blank_redirect_uri`. The default lambda permits blank URIs
when the application's grant flows exclude `authorization_code` and `implicit`. This requires
dropping the `NOT NULL` constraint on the redirect URI database column.

Use `use_url_path_for_native_authorization` to change the native authorization response route
from `oauth/authorize/native?code=<code>` to `oauth/authorize/<code>`.

## Application Owner

Register an owner for each OAuth application so you can scope tokens and authorizations per-user:

```ruby
Doorkeeper.configure do
  enable_application_owner confirmation: true
end
```

This requires running the `application_owner` generator (which adds a migration). When `confirmation: true`
is set (or `confirm_application_owner` is called), Doorkeeper enforces that every application has an owner.

## Token and Client Authentication Methods

Doorkeeper extracts the access token from the request using a configurable chain of methods:

```ruby
Doorkeeper.configure do
  access_token_methods :from_bearer_authorization, :from_access_token_param, :from_bearer_param
end
```

By default it checks the `Authorization: Bearer <token>` header first, then the `access_token` parameter,
then the `bearer_token` parameter. Client credentials are extracted similarly:

{% hint style="warning" %}
**Deprecated in Doorkeeper 6.0:** The `client_credentials` option below is deprecated. In 6.0+ use the new `client_authentication` option instead (see [Client Authentication](../ruby-on-rails/grant-flows.md#client-authentication)). The legacy syntax still works during the deprecation window — Doorkeeper automatically converts `:from_basic` to `client_secret_basic` and `:from_params` to `client_secret_post` — but it will be removed in a future major release.
{% endhint %}

```ruby
Doorkeeper.configure do
  client_credentials :from_basic, :from_params
end
```

This checks HTTP Basic Auth first (`client_id:client_secret`), then the `client_id` and `client_secret`
POST parameters. See [Token and Application Secrets](../security/token-and-application-secrets.md)
for hashing and secret storage strategies.

## Error Handling and Content-Type Enforcement

The `handle_auth_errors` option controls how Doorkeeper responds to authentication failures:

- `:render` (default) — returns a JSON error body with the appropriate HTTP status.
- `:raise` — raises `Doorkeeper::Errors::InvalidToken`, `Doorkeeper::Errors::TokenExpired`,
  `Doorkeeper::Errors::TokenRevoked`, and other exceptions for you to rescue in your application.
- `:redirect` — follows RFC 6749 section 4.1.2.1 redirect behaviour for authorization errors.

```ruby
Doorkeeper.configure do
  handle_auth_errors :raise
end
```

The `realm` option sets the `WWW-Authenticate` header realm value (default `"Doorkeeper"`).

Enforce the OAuth 2.0 content-type requirement for token endpoints:

```ruby
Doorkeeper.configure do
  enforce_content_type
end
```

This rejects requests that do not use `application/x-www-form-urlencoded`.

## Token Introspection

Enable and customize [RFC 7662](https://tools.ietf.org/html/rfc7662) token introspection:

```ruby
Doorkeeper.configure do
  allow_token_introspection do |token, authorized_client, authorized_token|
    authorized_client&.id == token&.application_id
  end

  custom_introspection_response do |token, context|
    { sub: token.resource_owner_id, aud: token.application&.uid }
  end
end
```

The default `allow_token_introspection` lambda permits introspection when the requesting token
belongs to the same application, or the authorized client matches the token's application, or
the token has no associated application. See [Token Introspection](token-introspection.md).

## Multi-Database Support

When using Rails read replicas with automatic role switching, enable this option to ensure
Doorkeeper write operations target the primary database:

```ruby
Doorkeeper.configure do
  enable_multiple_database_roles
end
```

This prevents `ActiveRecord::ReadOnlyError` on read replicas (Rails 6.1+).

## Lifecycle Hooks

Doorkeeper provides four lifecycle hooks. The authorization hooks fire during the browser-based
authorization flow:

```ruby
Doorkeeper.configure do
  before_successful_authorization do |controller, context|
    # e.g. single sign-out, audit logging
    Rails.logger.info "Authorizing #{context[:pre_auth]&.client&.name}"
  end

  after_successful_authorization do |controller, context|
    # context includes the issued token
    token = context[:issued_token]
    Rails.logger.info "Issued token #{token.id}"
  end
end
```

The strategy-response hooks fire during token endpoint requests:

```ruby
Doorkeeper.configure do
  before_successful_strategy_response do |request|
    # inspect/modify the request before the response is built
  end

  after_successful_strategy_response do |request, response|
    # response.body is the JSON about to be returned
    Rails.logger.info "Token response: #{response.body}"
  end
end
```

See also: [Scopes](scopes.md), [Models](models.md), [Token Introspection](token-introspection.md), [Token Revocation](token-revocation.md), [PKCE Flow](../ruby-on-rails/pkce-flow.md), [Token and Application Secrets](../security/token-and-application-secrets.md).
