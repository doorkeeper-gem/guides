# Access Grant

An Access Grant represents an **authorization code** issued during the Authorization Code flow. It is a short-lived, one-time-use credential that a client exchanges for an access token at the token endpoint. In Doorkeeper, access grants are stored in the `oauth_access_grants` table and modeled by `Doorkeeper::AccessGrant`.

## Key attributes

| Attribute | Description |
|---|---|
| `token` | The authorization code string sent to the client via the redirect URI. |
| `resource_owner_id` | The ID of the user who authorized the application. |
| `application_id` | The OAuth application (client) that requested authorization. |
| `redirect_uri` | The redirect URI used for this specific authorization request (must match one registered on the application). |
| `expires_in` | Time-to-live in seconds. Default is 600 (10 minutes), configurable via `authorization_code_expires_in`. |
| `scopes` | The scopes granted by the resource owner for this authorization. |
| `revoked_at` | Timestamp when the grant was revoked (if applicable). |

## PKCE fields

When [PKCE](../ruby-on-rails/pkce-flow.md) is enabled, two additional columns store the code challenge:

| Attribute | Description |
|---|---|
| `code_challenge` | The code challenge value sent during the authorization request. |
| `code_challenge_method` | The transformation method used (`plain` or `S256`). |

## Lifecycle

1. The resource owner approves an authorization request at `/oauth/authorize`.
2. Doorkeeper creates an `AccessGrant` record with a generated `token` (the authorization code).
3. The client is redirected to its `redirect_uri` with the code as a query parameter.
4. The client exchanges the code at `POST /oauth/token` with `grant_type=authorization_code`.
5. Doorkeeper verifies the code, revokes the grant, and issues an [Access Token](access-token.md).

Authorization codes are **one-time use** — once exchanged for a token, the grant is revoked and cannot be reused.

## Configuration

```ruby
Doorkeeper.configure do
  # TTL for authorization codes (default: 600 seconds / 10 minutes)
  authorization_code_expires_in 600
end
```

## See also

- [Database Design](../internals/database-design.md) — schema details for `oauth_access_grants`
- [Grant Flows](../ruby-on-rails/grant-flows.md) — the Authorization Code flow explained
- [PKCE Flow](../ruby-on-rails/pkce-flow.md) — securing the code exchange for public clients
