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

### Response Modes

The Authorization Code flow supports three response modes that control how the authorization code is delivered back to the client:

| Mode | Delivery method | Default |
|---|---|---|
| `query` | Appended as query parameters to the redirect URI | Yes |
| `fragment` | Appended as a URI fragment (`#code=...`) | No |
| `form_post` | Delivered via an auto-submitting HTML form POST to the redirect URI | No |

{% hint style="info" %}
The `fragment` and `form_post` response modes were introduced in Doorkeeper 5.5. Users on earlier versions only have `query` available for the Authorization Code flow.
{% endhint %}

To request a specific response mode, include `response_mode` in the authorization request (e.g., `response_mode=form_post`).

{% tabs %}
{% tab title="config/initializers/doorkeeper.rb" %}
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
{% endtab %}
{% endtabs %}

---

## Client Credentials

The Client Credentials flow is used for machine-to-machine communication where no resource owner is involved. The client authenticates with its own `client_id` and `client_secret` and requests a token with `grant_type=client_credentials` at `/oauth/token`.

Enabled by default.

**Related config options:**

- `revoke_previous_client_credentials_token` — when enabled, any existing non-expired token for the same application (and same `resource`, when RFC 8707 Resource Indicators are in use) is revoked before issuing a new one.
- `resource_indicator_validator` — when configured, enables [RFC 8707](https://datatracker.ietf.org/doc/html/rfc8707) support so that tokens issued via client credentials can be audience-restricted to specific resource servers.

{% tabs %}
{% tab title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper.configure do
  # Enabled by default. Optionally revoke previous tokens:
  revoke_previous_client_credentials_token

  # Optional: restrict tokens to specific resource servers (RFC 8707)
  # resource_indicator_validator do |resource_indicators, client|
  #   resource_indicators.all? { |r| allowed_apis.include?(r) }
  # end
end
```
{% endtab %}
{% endtabs %}

---

## Implicit (Deprecated)

The Implicit flow was designed for browser-based clients (SPAs) where the access token is returned directly in the URL fragment of the redirect URI, skipping the authorization code exchange step.

{% hint style="danger" %}
**Security caveat (RFC 6819, Section 4.4.2):** The access token is exposed in the URL fragment, making it susceptible to leakage through browser history, referrer headers, and JavaScript access. There is no client authentication step. This flow is **not recommended for new applications**. Use the Authorization Code flow with PKCE instead.
{% endhint %}

### Response Modes

The Implicit flow supports two response modes:

| Mode | Delivery method | Default |
|---|---|---|
| `fragment` | Appended as a URI fragment (`#access_token=...`) | Yes |
| `form_post` | Delivered via an auto-submitting HTML form POST to the redirect URI | No |

{% hint style="info" %}
The `form_post` response mode was introduced in Doorkeeper 5.5. Users on earlier versions only have `fragment` available for the Implicit flow.
{% endhint %}

To enable, add `"implicit"` to the `grant_flows` array:

{% tabs %}
{% tab title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper.configure do
  grant_flows %w[authorization_code client_credentials implicit]
end
```
{% endtab %}
{% endtabs %}

---

## Password (Resource Owner Password Credentials, Deprecated)

The Password flow allows a client to collect the resource owner's username and password directly and exchange them for an access token with `grant_type=password` at `/oauth/token`.

{% hint style="danger" %}
**Security caveat (RFC 6819, Section 4.4.3):** The client sees the user's raw credentials, which defeats the purpose of OAuth — the resource owner must trust the client with their password. This flow is **not recommended for new applications**. Prefer the Authorization Code flow.
{% endhint %}

Enabling this flow requires two pieces of configuration:

1. Add `"password"` to `grant_flows`.
2. Provide a `resource_owner_from_credentials` block that looks up and authenticates the user from the submitted username and password.

{% hint style="warning" %}
**Client authentication is required.** Since Doorkeeper 5.x, the Password grant requires valid client credentials (via HTTP Basic or the configured client authentication method). You may set `skip_client_authentication_for_password_grant` to `true` to allow public clients to use this flow, but this violates the OAuth spec and is discouraged. This option may be removed in a future major version.
{% endhint %}

{% tabs %}
{% tab title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper.configure do
  grant_flows %w[authorization_code client_credentials password]

  resource_owner_from_credentials do |_routes|
    User.find_by(email: params[:username])&.authenticate(params[:password])
  end

  # Optional: allow public clients to use this flow without a secret
  # (NOT recommended — violates OAuth spec)
  # skip_client_authentication_for_password_grant true
end
```
{% endtab %}
{% endtabs %}

---

## Refresh Token

The Refresh Token flow allows a client to exchange a refresh token for a new access token without re-involving the resource owner. The client sends `grant_type=refresh_token` along with the refresh token to `/oauth/token`.

This flow is **opt-in** via the `use_refresh_token` configuration method. It is not enabled by default.

When you enable refresh tokens, Doorkeeper issues a `refresh_token` alongside every access token (across all grant flows that produce tokens). The optional block passed to `use_refresh_token` receives a context object with `client`, `grant_type`, and `scopes` and can return `true` or `false` to decide whether a refresh token should be issued for that specific request.

{% hint style="info" %}
**Automatic grant flow registration:** As of Doorkeeper 6.0, enabling `use_refresh_token` automatically registers the `refresh_token` grant flow in `calculate_grant_flows`. You do **not** need to add `"refresh_token"` to the `grant_flows` array explicitly. If you do list it explicitly without enabling `use_refresh_token`, a configuration-time warning is logged (since no refresh tokens would ever be issued).
{% endhint %}

### Refresh Token Rotation

Refresh tokens are always rotated: every successful refresh request issues a new access token with a new refresh token, and the old refresh token is revoked. What the `previous_refresh_token` column on the access token model controls is **when** the old refresh token is revoked:

* **With the column** (`refresh_token_revoked_on_use?` returns `true`): the old refresh token stays valid until the newly issued access token is used for the first time, and is revoked at that point. This gives clients a grace window to retry a refresh whose response was lost (e.g. a network error) without being locked out. This is the default — the install migration generated by `rails generate doorkeeper:migration` creates the column.
* **Without the column**: the old refresh token is revoked immediately, inside the refresh request. Reusing it afterwards fails with `invalid_grant`. You get this behaviour by commenting out the `t.string :previous_refresh_token` line in the install migration, or on an application whose schema predates Doorkeeper 4.0 and never added the column.

If your schema is missing the column and you want the graceful behaviour, generate and run the previous-refresh-token migration:

```text
bundle exec rails generate doorkeeper:previous_refresh_token
bundle exec rails db:migrate
```

The `previous_refresh_token` value is stored on the access token record only; it is never included in the token response body.

{% tabs %}
{% tab title="config/initializers/doorkeeper.rb" %}
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
{% endtab %}
{% endtabs %}

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

{% hint style="info" %}
**Duplicate registration warning:** As of Doorkeeper 6.0, re-registering a flow that already exists emits a `[DOORKEEPER]` warning with the caller location. This helps catch accidental double-registrations from initializers or extensions.
{% endhint %}

You can also register aliases that expand into one or more existing flow names via `Doorkeeper::GrantFlow.register_alias`. This is useful when a single configuration name should enable multiple flows at once.

{% tabs %}
{% tab title="config/initializers/doorkeeper.rb" %}
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
{% endtab %}
{% endtabs %}

---

## Per-Client Flow Control

Doorkeeper provides two configuration blocks for controlling which flows a specific OAuth application may use and whether a resource owner is authorized to grant access to a given client.

### allow_grant_flow_for_client

This block receives the grant flow name (as a string) and the client (`Doorkeeper::Application` instance). Return `true` to allow the client to use that flow, or `false` to reject with `unauthorized_client`. By default, all clients may use all enabled grant flows.

{% tabs %}
{% tab title="config/initializers/doorkeeper.rb" %}
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
{% endtab %}
{% endtabs %}

### authorize_resource_owner_for_client

This block receives the client (`Doorkeeper::Application`) and the resource owner (your user model instance). Return `true` to authorize the owner for that client, or `false` to deny. By default, all owners are authorized for all clients.

{% hint style="info" %}
As of Doorkeeper 6.0, a denied `authorize_resource_owner_for_client` returns the correct `access_denied` error (per RFC 6749 Section 4.1.2.1) rather than the previously incorrect `invalid_client`.
{% endhint %}

{% tabs %}
{% tab title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper.configure do
  authorize_resource_owner_for_client do |client, resource_owner|
    # Example: only allow access if the user's organization matches the app's
    resource_owner.organization_id == client.organization_id
  end
end
```
{% endtab %}
{% endtabs %}

### Scopes by Grant Type

You can restrict which scopes are available to each grant type using the `scopes_by_grant_type` option. For details, see [Scopes](../configuration/scopes.md).

---

## Client Authentication

Doorkeeper 6.0 introduces a pluggable client authentication registry. The new `client_authentication` config option replaces the deprecated `client_credentials` option.

{% hint style="info" %}
**Version note:** On Doorkeeper < 6.0, client credential extraction is configured with the legacy `client_credentials :from_basic, :from_params` syntax. If you are running an older version, see the [legacy client credentials configuration](../configuration/other-configurations.md#token-and-client-authentication-methods) for details. The legacy option continues to work in 6.0 with automatic conversion but emits a deprecation warning at boot.
{% endhint %}

### Built-in Methods

| Method | Description |
|---|---|
| `client_secret_basic` | Client sends credentials via HTTP Basic Auth header (RFC 6749 §2.3.1). |
| `client_secret_post` | Client sends `client_id` and `client_secret` in the request body. |
| `none` | Public client authentication — only `client_id` is required, no secret. |
| `private_key_jwt` | Client authenticates with a signed JWT assertion (RFC 7523 / OIDC Core §9). Requires the `jwt` gem >= 2.7. |

{% hint style="warning" %}
**Breaking change in 6.0:** Client credentials are no longer read from the query string. Send them in the request body or via HTTP Basic. This applies to token, revocation, and introspection endpoints.
{% endhint %}

{% hint style="warning" %}
**Since 5.9.5:** Requests that use more than one client authentication method are rejected with `invalid_request` per RFC 6749 §2.3. This applies to all endpoints that authenticate clients (token, revocation, and introspection).
{% endhint %}

{% tabs %}
{% tab title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper.configure do
  # Declare accepted client authentication methods and their order:
  client_authentication %i[client_secret_basic client_secret_post none]

  # Or enable private_key_jwt for enhanced security:
  # client_authentication %i[client_secret_basic private_key_jwt]
end
```
{% endtab %}
{% endtabs %}

### Registering Custom Authentication Methods

You can register custom client authentication methods using `Doorkeeper::ClientAuthentication.register`:

{% tabs %}
{% tab title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper::ClientAuthentication.register(
  :my_custom_auth,
  # ... strategy implementation
)

Doorkeeper.configure do
  client_authentication %i[client_secret_basic my_custom_auth]
end
```
{% endtab %}
{% endtabs %}

### private_key_jwt

The `private_key_jwt` method (RFC 7523 / OIDC Core §9) allows clients to authenticate by sending a signed JWT assertion. The client's public keys are verified against `jwks` or `jwks_uri` attributes on the Application model.

Configuration options:

- `private_key_jwt_replay_guard` — jti single-use tracking. Defaults to a process-local store; supply a shared store for multi-process deployments (must respond to `first_use?(key, expires_at:)`).
- `private_key_jwt_jwks_cache` — cache for JWK Sets fetched from `jwks_uri`. Defaults to a process-local cache with 60-second TTL.

The audiences accepted for an assertion (`aud`) are built from the server's own identity — the `issuer` option or `Rails.application.routes.default_url_options[:host]` — never from the request's `Host` header. Configure at least one of them; if neither is set, every assertion is refused and Doorkeeper logs an error at boot.

---

## Authorization Server Metadata (RFC 8414)

Doorkeeper 6.0 exposes an OAuth 2.0 Authorization Server Metadata endpoint at `/.well-known/oauth-authorization-server`. The response is built from your configuration and advertises:

- Authorization, token, revocation, and introspection endpoints
- Supported scopes, response types, grant types, and PKCE code challenge methods
- `token_endpoint_auth_methods_supported` (derived from your `client_authentication` config)
- `authorization_response_iss_parameter_supported` (when `issuer` is configured, per RFC 9207)

{% tabs %}
{% tab title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper.configure do
  # Set an explicit issuer (defaults to request base URL)
  issuer "https://auth.example.com"

  # Merge custom data into the metadata response
  custom_metadata(
    userinfo_endpoint: "https://auth.example.com/userinfo"
  )
end
```
{% endtab %}
{% endtabs %}

### Issuer Identification (RFC 9207)

When `issuer` is configured, Doorkeeper adds the `iss` parameter to authorization responses (both successful and error redirects). This helps clients verify the response origin and protect against mix-up attacks.

---

## Further Reading

- [PKCE Flow](pkce-flow.md) — the recommended flow for public clients
- [Routes](routes.md) — mounted endpoints for `/oauth/authorize` and `/oauth/token`
- [Configuration overview](configuration.md)
- [Resource Indicators (RFC 8707)](../configuration/resource-indicators.md)
- [Other Configurations](../configuration/other-configurations.md)
