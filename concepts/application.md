# Application

An OAuth 2.0 Application (also called a "client") represents a third-party service that wants to access resources on behalf of a resource owner (user). In Doorkeeper, applications are stored in the `oauth_applications` table and modeled by `Doorkeeper::Application`.

## Key attributes

| Attribute | Description |
|---|---|
| `name` | Human-readable name displayed on the authorization screen. |
| `uid` | The **client identifier** — a public string used to identify the application in OAuth requests (`client_id`). Auto-generated on creation. |
| `secret` | The **client secret** — used together with `uid` to authenticate the client. Auto-generated on creation. |
| `redirect_uri` | One or more URIs the authorization server redirects the resource owner back to after granting/denying access. Multiple URIs are separated by newlines. |
| `scopes` | The set of scopes this application is allowed to request. |
| `confidential` | Boolean indicating whether the client can keep its secret safe. Public clients (mobile apps, SPAs) should set this to `false`. |

## Confidential vs. public clients

- **Confidential clients** (default, `confidential: true`) can securely store a client secret. They authenticate with both `client_id` and `client_secret` when exchanging authorization codes for tokens.
- **Public clients** (`confidential: false`) cannot guarantee secret confidentiality (e.g., native mobile apps, single-page JavaScript applications). They should use [PKCE](../ruby-on-rails/pkce-flow.md) to protect the authorization code flow.

## Creating applications

Applications can be created through the built-in admin UI at `/oauth/applications` (if enabled), via the Rails console, or programmatically:

```ruby
Doorkeeper::Application.create!(
  name: "My Client App",
  redirect_uri: "https://example.com/callback",
  scopes: "read write",
  confidential: true
)
```

In [API mode](../ruby-on-rails/api-mode.md), the admin UI is not available, so applications must be created via console or seeds.

## Application ownership

By default, applications do not belong to a user. Enable the `enable_application_owner` configuration to associate each application with a resource owner. See [Models](../configuration/models.md) for details.

## See also

- [Database Design](../internals/database-design.md) — schema details for `oauth_applications`
- [Token and Application Secrets](../security/token-and-application-secrets.md) — hashing strategies for the `secret` column
- [Models](../configuration/models.md) — custom model classes and ownership
