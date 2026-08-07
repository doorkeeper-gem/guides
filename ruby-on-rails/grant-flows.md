# Grant Flows

Doorkeeper supports the standard OAuth 2.0 grant flows as defined in [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749). A grant flow (also called a grant type) defines the sequence of steps an application performs to obtain an access token. By default, Doorkeeper enables only the **Authorization Code** and **Client Credentials** flows. The Implicit and Password flows are available but deprecated and must be explicitly enabled. The Refresh Token flow requires an explicit opt-in as well.

## Grant Flows Overview

| Flow | Default-enabled | Resource owner involved | Recommended |
|---|---|---|---|
| Authorization Code | Yes | Yes | Yes |
| Client Credentials | Yes | No (machine-to-machine) | Yes |
| Implicit | No | Yes | No (deprecated) |
| Password | No | Yes (credentials shared) | No (deprecated) |
| Refresh Token | No (opt-in) | No | Yes (use with rotation) |

---

## Authorization Code

The Authorization Code flow is the standard server-side OAuth flow. The client redirects the resource owner to `/oauth/authorize`, the owner approves, and Doorkeeper issues an authorization code. The client then exchanges the code for an access token at `/oauth/token` with `grant_type=authorization_code`.

This flow is enabled by default. It is the recommended choice for web applications with a backend that can securely store a client secret.

**Related config options:** `force_pkce`, `revoke_previous_authorization_code_token`.

For public clients (mobile apps, SPAs) that cannot keep a client secret, use the PKCE extension instead. See [PKCE Flow](pkce-flow.md).

{% code-tabs %}
{% code-tabs-item title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper.configure do
  # Authorization code is enabled by default — no config needed.
  # Optionally revoke the previous token when a new one is issued
  # for the same authorization code:
  revoke_previous_authorization_code_token

  # Require PKCE for public clients:
  force_pkce
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

---

## Client Credentials

The Client Credentials flow is used for machine-to-machine communication where no resource owner is involved. The client authenticates with its own `client_id` and `client_secret` and requests a token with `grant_type=client_credentials` at `/oauth/token`.

Enabled by default.

**Related config option:** `revoke_previous_client_credentials_token` — when enabled, any existing non-expired token for the same application is revoked before issuing a new one.

{% code-tabs %}
{% code-tabs-item title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper.configure do
  # Enabled by default. Optionally revoke previous tokens:
  revoke_previous_client_credentials_token
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

---

## Implicit (Deprecated)

The Implicit flow was designed for browser-based clients (SPAs) where the access token is returned directly in the URL fragment of the redirect URI, skipping the authorization code exchange step.

{% hint style="danger" %}
**Security caveat (RFC 6819, Section 4.4.2):** The access token is exposed in the URL fragment, making it susceptible to leakage through browser history, referrer headers, and JavaScript access. There is no client authentication step. This flow is **not recommended for new applications**. Use the Authorization Code flow with PKCE instead.
{% endhint %}

To enable, add `"implicit"` to the `grant_flows` array:

{% code-tabs %}
{% code-tabs-item title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper.configure do
  grant_flows %w[authorization_code client_credentials implicit]
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

---

## Password (Resource Owner Password Credentials, Deprecated)

The Password flow allows a client to collect the resource owner's username and password directly and exchange them for an access token with `grant_type=password` at `/oauth/token`.

{% hint style="danger" %}
**Security caveat (RFC 6819, Section 4.4.3):** The client sees the user's raw credentials, which defeats the purpose of OAuth — the resource owner must trust the client with their password. This flow is **not recommended for new applications**. Prefer the Authorization Code flow.
{% endhint %}

Enabling this flow requires two pieces of configuration:

1. Add `"password"` to `grant_flows`.
2. Provide a `resource_owner_from_credentials` block that looks up and authenticates the user from the submitted username and password.

You may also set `skip_client_authentication_for_password_grant` to `true` if you wish to allow public clients (without a client secret) to use this flow.

{% code-tabs %}
{% code-tabs-item title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper.configure do
  grant_flows %w[authorization_code client_credentials password]

  resource_owner_from_credentials do |_routes|
    User.find_by(email: params[:username])&.authenticate(params[:password])
  end

  # Optional: allow public clients to use this flow without a secret
  # skip_client_authentication_for_password_grant true
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

---

## Refresh Token

The Refresh Token flow allows a client to exchange a refresh token for a new access token without re-involving the resource owner. The client sends `grant_type=refresh_token` along with the refresh token to `/oauth/token`.

This flow is **opt-in** via the `use_refresh_token` configuration method. It is not enabled by default.

When you enable refresh tokens, Doorkeeper issues a `refresh_token` alongside every access token (across all grant flows that produce tokens). The optional block passed to `use_refresh_token` receives a context object with `client`, `grant_type`, and `scopes` and can return `true` or `false` to decide whether a refresh token should be issued for that specific request.

### Refresh Token Rotation

Doorkeeper supports refresh token rotation: when a refresh token is used, the old one can be revoked and a new refresh token issued in its place. This is controlled by the `previous_refresh_token` column on the access token model. To enable rotation, generate and run the previous-refresh-token migration:

```text
bundle exec rails generate doorkeeper:previous_refresh_token
bundle exec rails db:migrate
```

When rotation is active, each token response that includes a `refresh_token` also includes a `previous_refresh_token` field containing the refresh token that was just consumed.

{% code-tabs %}
{% code-tabs-item title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper.configure do
  grant_flows %w[authorization_code client_credentials]

  use_refresh_token

  # Optional: conditionally issue refresh tokens
  # use_refresh_token do |context|
  #   context.client.confidential?
  # end
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

---

## Custom Grant Flows

Doorkeeper allows registration of custom grant flows through the `Doorkeeper::GrantFlow.register` method. A custom flow can respond to a custom `grant_type`, a custom `response_type`, or both, and can define its own request strategy class.

The registration API is exposed via `Doorkeeper::GrantFlow::Registry` (module), which is extended onto `Doorkeeper::GrantFlow`. The `register` method accepts a flow name symbol plus keyword options:

| Option | Description |
|---|---|
| `grant_type_matches` | The `grant_type` parameter value this flow handles on the token endpoint (e.g., `"authorization_code"`). |
| `grant_type_strategy` | The request strategy class that processes token requests for this grant type. Must inherit from or implement the interface of `Doorkeeper::Request::Strategy`. |
| `response_type_matches` | The `response_type` parameter value this flow handles on the authorize endpoint (e.g., `"code"`). |
| `response_type_strategy` | The request strategy class that processes authorization requests for this response type. |
| `response_mode_matches` | An array of accepted response modes (e.g., `%w[query fragment form_post]`). |

You can also register aliases that expand into one or more existing flow names via `Doorkeeper::GrantFlow.register_alias`. This is useful when a single configuration name should enable multiple flows at once.

{% code-tabs %}
{% code-tabs-item title="config/initializers/doorkeeper.rb" %}
```ruby
# Register a custom grant flow
Doorkeeper::GrantFlow.register(
  :saml_bearer,
  grant_type_matches: "urn:ietf:params:oauth:grant-type:saml2-bearer",
  grant_type_strategy: SamlBearerRequestStrategy,
)

# Register an alias that enables multiple flows
Doorkeeper::GrantFlow.register_alias(
  :oidc,
  as: %w[authorization_code implicit],
)

Doorkeeper.configure do
  # Now you can reference the alias in grant_flows
  grant_flows %w[authorization_code client_credentials saml_bearer]
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

---

## Per-Client Flow Control

Doorkeeper provides two configuration blocks for controlling which flows a specific OAuth application may use and whether a resource owner is authorized to grant access to a given client.

### allow_grant_flow_for_client

This block receives the grant flow name (as a string) and the client (`Doorkeeper::Application` instance). Return `true` to allow the client to use that flow, or `false` to reject with `unauthorized_client`. By default, all clients may use all enabled grant flows.

{% code-tabs %}
{% code-tabs-item title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper.configure do
  allow_grant_flow_for_client do |grant_flow, client|
    # Example: only allow the client_credentials flow for confidential apps
    if grant_flow == "client_credentials"
      client.confidential?
    else
      true
    end
  end
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### authorize_resource_owner_for_client

This block receives the client (`Doorkeeper::Application`) and the resource owner (your user model instance). Return `true` to authorize the owner for that client, or `false` to deny. By default, all owners are authorized for all clients.

{% code-tabs %}
{% code-tabs-item title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper.configure do
  authorize_resource_owner_for_client do |client, resource_owner|
    # Example: only allow access if the user's organization matches the app's
    resource_owner.organization_id == client.organization_id
  end
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### Scopes by Grant Type

You can restrict which scopes are available to each grant type using the `scopes_by_grant_type` option. For details, see [Scopes](../configuration/scopes.md).

---

## Further Reading

- [PKCE Flow](pkce-flow.md) — the recommended flow for public clients
- [Routes](routes.md) — mounted endpoints for `/oauth/authorize` and `/oauth/token`
- [Configuration overview](configuration.md)
- [Other Configurations](../configuration/other-configurations.md)
