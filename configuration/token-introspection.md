# Token Introspection

Token introspection is the process defined by [RFC 7662](https://datatracker.ietf.org/doc/html/rfc7662) that allows resource servers (APIs) to validate access
tokens presented by clients. Instead of sharing cryptographic keys with every resource server, the resource server sends
the token to the authorization server's introspection endpoint and receives back metadata about the token's current 
state — whether it is active, who it was issued to, its scope, and when it expires. Doorkeeper implements this endpoint at `POST /oauth/introspect`.

## The Endpoint

**Route:** `POST /oauth/introspect`

Doorkeeper registers this route automatically unless `allow_token_introspection` is set to `false` (which removes the route entirely).
The endpoint requires authentication — the resource server must identify itself using either:

* **Client authentication** (HTTP Basic Auth with `client_id` and `client_secret`), or
* **Bearer token** (an OAuth 2.0 access token in the `Authorization: Bearer <token>` header).

The token to be introspected is sent as the `token` POST parameter. An optional `token_type_hint` parameter (`"access_token"`
or `"refresh_token"`) helps Doorkeeper locate the token efficiently.

### Request

```http
POST /oauth/introspect HTTP/1.1
Host: auth.example.com
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded

token=tGzv3JOkF0XG5Qx2TlKWIA&token_type_hint=access_token
```

### Response (Active Token)

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "active": true,
  "scope": "read write",
  "client_id": "abc123def456",
  "token_type": "Bearer",
  "iat": 1755009123,
  "exp": 1755016323
}
```

The response fields returned by Doorkeeper are:

| Field | Source | Notes |
|---|---|---|
| `active` | computed | `true` if the token exists, is not expired, is not revoked, and the requester is allowed to introspect it; `false` otherwise. |
| `scope` | `token.scopes_string` | Space-separated list of scopes granted to the token. |
| `client_id` | `token.application.uid` | The UID of the OAuth application that the token was issued to. Omitted if the token has no application. |
| `token_type` | `token.token_type` | `"Bearer"` for access tokens. |
| `iat` | `token.created_at.to_i` | Issued-at timestamp (Unix epoch seconds). |
| `exp` | `token.expires_at.to_i` | Expiration timestamp (Unix epoch seconds). Present only when the token has an expiry set; omitted for non-expiring tokens. |

### Response (Inactive or Unauthorized Token)

When the introspection request is properly authorized but the token is invalid, expired, revoked, or the requester is not
permitted to introspect it, Doorkeeper returns an `active: false` response with no additional metadata:

```json
{
  "active": false
}
```

This is per RFC 7662 Section 2.2 — the authorization server should not disclose why a token is inactive to avoid leaking state information to third parties.

---

## Configuration

### `allow_token_introspection`

Controls who is allowed to introspect a token. The default is an arity-3 lambda that permits introspection in three cases:

1. The authorized bearer token belongs to the same application as the token being introspected.
2. The authorized client (Basic auth) is the same application that the token was issued to.
3. The token has no associated application (public token).

```ruby
# Default behavior (arity 3 lambda):
#
# allow_token_introspection do |token, authorized_client, authorized_token|
#   if authorized_token
#     authorized_token.application_id == token&.application_id
#   elsif token&.application
#     authorized_client.id == token.application_id
#   else
#     true
#   end
# end
```

The block receives three arguments:

* `token` — `Doorkeeper::AccessToken` (the token being introspected; `nil` if using bearer auth and no matching token found)
* `authorized_client` — `Doorkeeper::Application` (the client authenticated via Basic auth; `nil` when using bearer token auth)
* `authorized_token` — `Doorkeeper::AccessToken` (the bearer token used to authorize the request; `nil` when using client auth)

Exactly one of `authorized_client` and `authorized_token` will be non-nil. Use nil-safe operators (`&.`) when calling methods on these arguments.

Returning `nil` or `false` from the block rejects introspection:
* With bearer token auth — returns HTTP 401.
* With client auth — returns HTTP 200 with `{ "active": false }` (per RFC 7662 Section 2.2).

To restrict introspection to specific trusted clients:

```ruby
Doorkeeper.configure do
  allow_token_introspection do |token, authorized_client, authorized_token|
    if authorized_token
      # Require the bearer token to have an 'introspection' scope
      authorized_token.scopes.include?("introspection")
    elsif token&.application
      # Only allow a specific trusted introspection client
      authorized_client.uid == ENV["INTROSPECTION_CLIENT_ID"]
    else
      false
    end
  end
end
```

To disable token introspection entirely:

```ruby
Doorkeeper.configure do
  allow_token_introspection false
end
```

Setting `false` removes the `POST /oauth/introspect` route completely. No introspection requests will be accepted.

### `custom_introspection_response`

Add custom fields to the introspection response JSON. The value must be a proc, lambda, or any object responding to `.call`
that returns a `Hash`. The default returns an empty hash:

```ruby
# Default:
# custom_introspection_response ->(_token, _context) { {} }
```

The block receives two arguments:

* `token` — the `Doorkeeper::AccessToken` being introspected
* `context` — the request context (the controller instance)

The returned hash is merged into the standard introspection response. Fields like `sub`, `aud`, and `username` are commonly
added to conform to RFC 7662's optional metadata:

```ruby
Doorkeeper.configure do
  custom_introspection_response do |token, context|
    {
      sub: token.resource_owner_id,
      aud: token.application&.uid,
      username: User.find_by(id: token.resource_owner_id)&.email
    }
  end
end
```

The resulting introspection response would then include the standard fields plus your custom additions:

```json
{
  "active": true,
  "scope": "read",
  "client_id": "abc123",
  "token_type": "Bearer",
  "iat": 1755009123,
  "exp": 1755016323,
  "sub": "42",
  "aud": "abc123",
  "username": "user@example.com"
}
```

---

**See also:** [Other Configurations](other-configurations.md), [Token Revocation](token-revocation.md), [Controllers & Helpers](../ruby-on-rails/controllers-and-helpers.md).
