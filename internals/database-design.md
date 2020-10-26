# Database Design

## oauth_applications

| Field | Purpose |
| :--- | :--- |
| **id** | Primary key, in case of using RDBMs |
| **name** | Application name |
| **uid** | Unique ID, used as [_client identifier_](https://tools.ietf.org/html/rfc6749#section-2.2) |
| **secret** | Used together with `uid` for client authentication |
| **redirect_uri** | Redirects the resource owner to this URI \([spec](https://tools.ietf.org/html/rfc6749#section-3.1.2)\) |
| **scopes** | Defines which [scopes](../configuration/scopes.md) the application uses |
| **confidential** | Indicates whether client public or private |
| **created_at** | Creation date & time |
| **updated_at** | Date & time of latest update |

If you set `enable_application_owner` configuration option then applications table also includes:

| Field | Purpose |
| :--- | :--- |
| **owner_id** | PK of the Resource owner record |
| **owner_type** | Resource owner model name |

## oauth_access_tokens

| Field | Purpose |
| :--- | :--- |
| **id** | Primary key, in case of using RDBMs |
| **resource_owner_id** | PK of the resource owner record |
| **application_id** | PK of the client token was issued for |
| **token** | Token value |
| **refresh_token** | Refresh token value (used to refresh a token) |
| **expires_in** | TTL of the token (in seconds) |
| **revoked_at** | Date & time when token was revoked |
| **created_at** | Creation date & time |
| **scopes** | Access token scopes |
| **previous_refresh_token**| Previous refresh token value |

If you enabled `use_polymorphic_resource_owner` configuration option then your database must
have additional columns:

| Field | Purpose |
| :--- | :--- |
| **resource_owner_type** | Resource owner model name |

## oauth_access_grants

| Field | Purpose |
| :--- | :--- |
| **id** | Primary key, in case of using RDBMs |
| **resource_owner_id** | PK of the resource owner record |
| **application_id** | PK of the client token was issued for |
| **token** | Token value |
| **expires_in** | TTL of the token (in seconds) |
| **redirect_uri** | Redirect URI |
| **revoked_at** | Date & time when token was revoked |
| **created_at** | Creation date & time |
| **scopes** | Access token scopes |

In case you enabled PKCE flow, your access grants table will include:

| Field | Purpose |
| :--- | :--- |
| **code_challenge** | Code challenge value |
| **code_challenge_method** | Code challenge method name |
