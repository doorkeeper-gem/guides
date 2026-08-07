# Token Revocation

Doorkeeper implements [RFC 7009 (OAuth 2.0 Token Revocation)](https://tools.ietf.org/html/rfc7009), allowing clients to explicitly invalidate access and refresh tokens. This is useful when a user logs out of an application or when a client needs to discard a token it no longer needs, preventing further use of that token.

## The Endpoint

Doorkeeper exposes a single `POST /oauth/revoke` route for token revocation, mapped to the `revoke` action on `Doorkeeper::TokensController`.

### Request Format

The request must include:

- **`token`** (required) — the token string to revoke.
- **`token_type_hint`** (optional) — either `access_token` or `refresh_token`, to help Doorkeeper locate the token more efficiently.
- **Client authentication** — the client must authenticate itself using one of the methods defined in RFC 6749 Section 2.3 (e.g., client ID and secret via HTTP Basic Auth or request body).

```http
POST /oauth/revoke HTTP/1.1
Host: example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW

token=45c9b8d7a1f3e0c2b6d8f1a3e5c7d9b0&token_type_hint=access_token
```

### Response

On success, Doorkeeper returns HTTP **200** with an empty body:

```http
HTTP/1.1 200 OK

{}
```

If the client is not authenticated, or if the token belongs to a different client, Doorkeeper returns HTTP 403:

```http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
  "error": "unauthorized_client",
  "error_description": "The client is not authorized to revoke this token."
}
```

## Idempotency

Since version 5.9.0 ([PR #1778](https://github.com/doorkeeper-gem/doorkeeper/pull/1778)), Doorkeeper revocation is **idempotent**. Revoking a token that has already been revoked, or submitting an unknown token, returns **HTTP 200** — the same response as a successful revocation — rather than an error.

The security rationale is that the endpoint should not leak whether a token exists or was previously valid. If an attacker could probe the endpoint and receive different responses for valid vs. invalid tokens, they could use revocation as an oracle to enumerate active tokens. By returning a consistent 200 response regardless, Doorkeeper follows the RFC 7009 recommendation.

Version 5.9.1 ([PR #1806](https://github.com/doorkeeper-gem/doorkeeper/pull/1806)) further tightened revocation for public clients to prevent a bypass under RFC 7009.

## Related Configuration

### `revoke_previous_client_credentials_token`

When enabled, issuing a new access token via the client credentials grant automatically revokes any previous access token for the same client. This enforces a "one valid token per client" policy.

```ruby
Doorkeeper.configure do
  revoke_previous_client_credentials_token
end
```

**Default:** disabled (false).

### `revoke_previous_authorization_code_token`

When enabled, issuing a new access token via the authorization code grant automatically revokes any previous access token for the same client and resource owner. This enforces a "one valid token per client/resource-owner pair" policy.

```ruby
Doorkeeper.configure do
  revoke_previous_authorization_code_token
end
```

**Default:** disabled (false).

## See Also

- [Other Configurations](other-configurations.md)
- [Token Introspection](token-introspection.md)
- [Controllers & Helpers](../ruby-on-rails/controllers-and-helpers.md)
