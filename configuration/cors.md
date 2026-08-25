# Configuring CORS

Doorkeeper does not handle CORS (Cross-Origin Resource Sharing) itself. Browser-based clients
that call your OAuth-protected API from a different origin need CORS headers on the response,
otherwise the browser blocks the request. You configure CORS at the Rack middleware level using
the [`rack-cors`](https://github.com/cyu/rack-cors) gem.

## Why CORS matters for Doorkeeper

Browser and single-page application (SPA) clients interact with Doorkeeper endpoints in several
ways, and CORS requirements differ for each:

- **Protected API resources** — When a SPA on `https://app.example.com` calls your API on
  `https://api.example.com` with `Authorization: Bearer <token>`, the browser sends a
  cross-origin request. Without CORS headers the browser blocks the response entirely.
- **Token endpoint (`POST /oauth/token`)** — SPAs using Authorization Code + PKCE exchange the
  authorization code for a token directly from the browser. This is an XHR/fetch call to
  `/oauth/token` and needs CORS.
- **Authorization endpoint (`GET /oauth/authorize`)** — This is a full-page redirect or
  navigation, not an XHR request, so CORS is generally not required for this endpoint.
- **Introspection (`POST /oauth/introspect`) and revocation (`POST /oauth/revoke`)** — These
  are typically server-to-server calls. You should avoid exposing them to browser origins
  without a specific reason.

## Setup with rack-cors

Add the gem to your `Gemfile` and install it:

```ruby
gem 'rack-cors'
```

```
bundle install
```

Next, create the CORS initializer:

{% tabs %}
{% tab title="config/initializers/cors.rb" %}
```ruby
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins "https://app.example.com", "http://localhost:3000"

    resource "*",
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head]
  end
end
```
{% endtab %}
{% endtabs %}

Place the middleware at position `0` (before all others) so that CORS headers are added on every
response, including error responses from deeper in the stack.

### Scoping resources

Where practical, scope the `resource` directive to the paths that actually need CORS instead of
using `"*"`:

{% tabs %}
{% tab title="config/initializers/cors.rb" %}
```ruby
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins "https://app.example.com"

    resource "/api/*",
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head]

    resource "/oauth/token",
      headers: :any,
      methods: [:post, :options]
  end
end
```
{% endtab %}
{% endtabs %}

Scoping limits which paths are reachable cross-origin and is a good security practice.

## Headers and credentials

### The Authorization header

Bearer tokens are sent in the `Authorization` header. The `headers: :any` setting in the examples
above permits all headers, including `Authorization`. This is the critical header to allow for
Doorkeeper-protected APIs. If you restrict headers to a specific list, you must include
`"Authorization"` explicitly.

If your clients also use HTTP Basic authentication for the client credentials flow (sending
`client_id` and `client_secret` in the `Authorization` header as `Basic <base64>`), the same
`headers: :any` setting covers that as well.

### Credentials

The `credentials: true` option tells the browser to include cookies and HTTP authentication on
cross-origin requests. It is needed **only** when your requests carry cookies (for example,
session-based or cookie-based authentication alongside OAuth).

For pure Bearer-token API access — where the token is sent in the `Authorization` header and
no cookies are involved — credentials are **not** required. Leave the default (`false`).

A common mistake is enabling `credentials: true` unnecessarily, or worse, combining it with
a wildcard origin:

```ruby
# BROKEN — browsers reject origins: "*" with credentials: true
allow do
  origins "*"
  resource "*", headers: :any, methods: [:get, :post], credentials: true
end
```

The browser will silently reject this combination. When you need credentials (cookies), use
an explicit allowlist of origins:

```ruby
allow do
  origins "https://app.example.com", "https://admin.example.com"
  resource "*", headers: :any, methods: [:get, :post], credentials: true
end
```

### Preflight requests

Browsers send an `OPTIONS` preflight request before cross-origin requests that use non-simple
headers (such as `Authorization`). The `rack-cors` middleware handles these preflight requests
automatically, provided `:options` is included in the `methods` list — as shown in all examples
above.

## Security considerations

- **Prefer explicit origins.** Avoid `origins "*"` for token and API endpoints. List the
  specific, trusted origins that should be allowed to make cross-origin requests.
- **Never combine `origins "*"` with `credentials: true`.** Browsers reject this combination
  outright (it violates the CORS specification).
- **Do not expose introspection and revocation endpoints to browser origins** unless you have
  a specific, well-understood reason for doing so. These are typically server-to-server
  endpoints.
- **Scope resource paths.** Use `/api/*` and `/oauth/token` rather than `"*"` where possible to
  limit the surface area open to cross-origin access.
- **Keep origins aligned with your deployment.** Add production, staging, and local development
  origins explicitly. Review the list when your front-end deployment changes.

## Rails API-only applications

In Rails API-only applications the `rack-cors` gem works the same way. The initializer uses
`config.middleware.insert_before`, which inserts the CORS middleware regardless of whether the
app is a full Rails app or an API-only app. If the middleware is not taking effect, verify it
appears in the middleware stack:

```
bin/rails middleware
```

Also confirm Doorkeeper is configured for API mode. See [API Mode](../ruby-on-rails/api-mode.md).

## Common pitfalls

- **Forgetting to allow the `Authorization` header.** Without it, the browser strips the
  Bearer token from cross-origin requests and your API sees an unauthenticated request.
- **Using `origins "*"` with `credentials: true`.** The combination is a spec violation and
  browsers silently drop it — cross-origin requests fail with no clear error.
- **Not including `:options` in the methods list.** Preflight requests are not handled and
  the actual request never reaches your app.
- **CORS headers not appearing in Rails API-only mode.** Run `bin/rails middleware` to confirm
  `Rack::Cors` is in the stack. If it is missing, check that the initializer filename ends with
  `.rb` and is loaded.

## See also

- [API Mode](../ruby-on-rails/api-mode.md)
- [Securing the API](../ruby-on-rails/protecting-your-resources.md)
- [PKCE Flow](../ruby-on-rails/pkce-flow.md)
- [Other Configurations](other-configurations.md)
