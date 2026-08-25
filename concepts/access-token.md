# Access Token

An Access Token is the credential that a client uses to access protected resources on behalf of a resource owner. In Doorkeeper, access tokens are stored in the `oauth_access_tokens` table and modeled by `Doorkeeper::AccessToken`.

## Key attributes

| Attribute | Description |
|---|---|
| `token` | The access token string presented in API requests (via `Authorization: Bearer <token>` header). |
| `refresh_token` | An optional token used to obtain a new access token without re-involving the resource owner. Present when `use_refresh_token` is enabled. |
| `resource_owner_id` | The ID of the user who authorized the token. `nil` for client credentials grants. |
| `application_id` | The OAuth application (client) that the token was issued to. |
| `expires_in` | Time-to-live in seconds. Default is 7200 (2 hours), configurable via `access_token_expires_in`. |
| `scopes` | The scopes granted to this token. |
| `revoked_at` | Timestamp when the token was revoked (if applicable). |
| `previous_refresh_token` | Optional column. When present, the refresh token that was consumed to issue this token is stored here and only revoked once this token is used for the first time (graceful rotation). Without the column, the consumed refresh token is revoked immediately on refresh. |

## Token lifecycle

1. A client obtains a token through one of the [grant flows](../ruby-on-rails/grant-flows.md).
2. The client includes the token in API requests: `Authorization: Bearer <token>`.
3. The resource server validates the token using `doorkeeper_authorize!`.
4. When the token expires, the client uses the refresh token (if available) to get a new one.
5. Tokens can be explicitly revoked via `POST /oauth/revoke` ([Token Revocation](../configuration/token-revocation.md)).

## Token validation

A token is considered **valid** (active) when all of the following are true:

- It has not been revoked (`revoked_at` is `nil`).
- It has not expired (current time < `created_at + expires_in`).
- It has the required scopes for the requested resource.

## Configuration

```ruby
Doorkeeper.configure do
  # TTL for access tokens (default: 7200 seconds / 2 hours)
  access_token_expires_in 2.hours

  # Per-context TTL override
  custom_access_token_expires_in do |context|
    context.grant_type == Doorkeeper::OAuth::CLIENT_CREDENTIALS ? 1.hour : nil
  end

  # Issue refresh tokens alongside access tokens
  use_refresh_token

  # Reuse existing valid tokens instead of issuing new ones
  # reuse_access_token
end
```

## Token introspection

Resource servers can validate tokens without sharing cryptographic material by calling the introspection endpoint (`POST /oauth/introspect`). See [Token Introspection](../configuration/token-introspection.md).

## See also

- [Database Design](../internals/database-design.md) — schema details for `oauth_access_tokens`
- [Grant Flows](../ruby-on-rails/grant-flows.md) — how tokens are issued
- [Token and Application Secrets](../security/token-and-application-secrets.md) — hashing strategies for stored tokens
- [Token Revocation](../configuration/token-revocation.md) — revoking tokens via the API
- [Other Configurations](../configuration/other-configurations.md) — expiry, reuse, and generator options
