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

## Index recommendations

Doorkeeper's default migration template creates the following indexes. If you
maintain a manual schema, ensure these are present. All column names verified
against the 5.9.5 migration template
(`lib/generators/doorkeeper/templates/migration.rb.erb`).

**oauth_applications**

| Index | Type | Notes |
| :--- | :--- | :--- |
| `uid` | unique | Required for client lookup by UID |

**oauth_access_grants**

| Index | Type | Notes |
| :--- | :--- | :--- |
| `token` | unique | Authorization code lookup |
| `application_id` | foreign key | References `oauth_applications.id` |

**oauth_access_tokens**

| Index | Type | Notes |
| :--- | :--- | :--- |
| `token` | unique | Primary token lookup |
| `refresh_token` | unique | Refresh token lookup; on SQL Server this is a filtered unique index (`WHERE refresh_token IS NOT NULL`) |
| `application_id` | foreign key | References `oauth_applications.id` |
| `resource_owner_id` | B-tree | Created automatically via `t.references :resource_owner, index: true` |

For PostgreSQL you may prefer a partial unique index on `refresh_token`
(`WHERE refresh_token IS NOT NULL`) to allow multiple NULL values while
still enforcing uniqueness on non-null tokens.

## Token hashing storage columns

When `hash_token_secrets` or `hash_application_secrets` is enabled, hashed
values are stored **in the same columns** — no additional database columns are
required. The plaintext value is never persisted.

| Configuration option | Affected column(s) | Strategy |
| :--- | :--- | :--- |
| `hash_token_secrets` | `oauth_access_tokens.token`, `oauth_access_grants.token`, `oauth_access_tokens.refresh_token` | Plain, Sha256Hash, or BCrypt |
| `hash_application_secrets` | `oauth_applications.secret` | Plain, Sha256Hash, or BCrypt |

The `:plain` strategy is the fallback — it stores values as-is with no hashing.
Strategies are configured via `Doorkeeper.configure`:

```ruby
Doorkeeper.configure do
  hash_token_secrets
  hash_application_secrets using: '::Doorkeeper::SecretStoring::BCrypt'
end
```

See [Token and Application Secrets](../security/token-and-application-secrets.md)
for detailed configuration and security considerations.

## Migration notes

Doorkeeper added these columns incrementally across 5.x releases. Each
migration template is self-contained and safe to run on any version.

| Migration template | Table | Columns added | Doorkeeper version |
| :--- | :--- | :--- | :--- |
| `enable_pkce_migration.rb.erb` | `oauth_access_grants` | `code_challenge` (string), `code_challenge_method` (string) | 5.0.0 |
| `add_owner_to_application_migration.rb.erb` | `oauth_applications` | `owner_id` (bigint), `owner_type` (string) | 5.0.1 |
| `enable_polymorphic_resource_owner_migration.rb.erb` | `oauth_access_tokens`, `oauth_access_grants` | `resource_owner_type` (string) | 5.4.0 |
| `add_previous_refresh_token_to_access_tokens.rb.erb` | `oauth_access_tokens` | `previous_refresh_token` (string, null: false, default: "") | 5.4.0 |
| `add_confidential_to_applications.rb.erb` | `oauth_applications` | `confidential` (boolean, null: false, default: true) | 5.0.0 |
| `remove_applications_secret_not_null_constraint.rb.erb` | `oauth_applications` | Removes `NOT NULL` from `secret` | 5.8.0 |

All column names and types above were verified against the corresponding
migration templates in Doorkeeper 5.9.5
(`lib/generators/doorkeeper/templates/`).
