# Hashing of token and application secrets

Doorkeeper can optionally Hash access, refresh tokens, and application secrets before persisting them.
Since integrations may have decided on showing application secrets later on, both options can be enabled individually.

To enable hashing of access and refresh tokens, uncomment the initializer line `hash_token_secrets`.
Upon issuing a new token, the plain token value will be available for presenting to the user in `token.plaintext_token`.
Later comparisons will only be performed on the hashed token and the plain token can no longer be retrieved.

**Fallback to plain token lookup**

When you are upgrading doorkeeper and have existing access or refresh tokens in your database that you want to keep,
enabling `hash_token_secrets` would implicitly invalidate all plaintext tokens since the plain tokens would no longer be found.

To enable plain values to be found and upgraded to a hashed value whenever it is accessed, you may use the following statement:

```ruby
  hash_token_secrets fallback: :plain
```

This is only needed for _existing_ doorkeeper installations with plain tokens present.

**Other secret implementations**

By default, hashing of all secrets in doorkeeper will be performed using `Digest::SHA256`.
You can provide other secret transformer implementations. To that end, have a look at the `Doorkeeper::SecretStoring::Sha256Hash`
class. To specify another implementation, please use

```ruby
  hash_token_secrets using: '::My::Other::SecretStoring::Implementation'
```

Please note that if you used another strategy such as SHA256 or plain storage in the past, you always need to specify
that in the `fallback:` option, otherwise all tokens stored under the previous secret storage implementation will be invalid.

**Incompatibility with reuse_access_token**

Enabling `hash_token_secrets` is incompatible with the option `reuse_access_token`
since plain values can no longer be retrieved. If you enabled both,
the latter will be disabled with a warning.

To enable hashing application secrets (`client_secret`), uncomment the initializer line `hash_application_secrets`.
Application secrets will then by hashed by `Digest::SHA256` before saving.

This will result in the `secret` value of the application being available during the request that created in as `application.plaintext_secret`.
In this request, you need to ensure the user noted the secret since you will no longer be able to show it afterwards.


**Using BCrypt gem for application secrets**

Since application secrets are to be treated as password, Doorkeeper also allows you to store secrets as BCrypt hashes.
To enable it, simply add it to your Gemfile **This will add ~200ms latency to endpoints that verify client secret**: 

{% tabs %}
{% tab title="Gemfile" %}
```ruby
gem 'doorkeeper'
gem 'bcrypt', require: false
```
{% endtab %}
{% endtabs %}

and then use the following configuration instead:

```ruby
  hash_application_secrets using: '::Doorkeeper::SecretStoring::BCrypt'
```

**Fallback of plain application secrets**

When you are upgrading doorkeeper and have existing application secrets in your database that you want to upgrade,
enabling `hash_application_secrets` would implicitly invalidate all plaintext secrets since they would no longer be found.

To enable plain values to be found and upgraded (to your active strategy, SHA256 by default) when it is accessed, you may use the following statement:

```ruby
  hash_application_secrets fallback: :plain
```

This is only needed for _existing_ doorkeeper installations with plain application secrets present.


## Default behavior

By default, Doorkeeper stores all token and application secrets in **plaintext**. Neither `hash_token_secrets` nor `hash_application_secrets`
is enabled out of the box -- both options are commented out in the initializer template. Internally, this means `token_secret_strategy` and
`application_secret_strategy` both fall back to `::Doorkeeper::SecretStoring::Plain`.

Hashing is strictly opt-in. You must uncomment and configure the corresponding initializer lines to enable it. If you never touch these settings,
secrets are stored exactly as generated and can be retrieved from the database at any time.

```ruby
# In config/initializers/doorkeeper.rb -- BOTH lines are commented by default:
# hash_token_secrets
# hash_application_secrets
```


## Secret storing strategies

Doorkeeper provides three built-in secret storing strategies, all under the `Doorkeeper::SecretStoring` namespace.
Each strategy must implement `transform_secret` (to produce a storable value) and `allows_restoring_secrets?` (whether the
original plaintext can be recovered from the stored value).

### Plain (default)

`Doorkeeper::SecretStoring::Plain` stores secrets as-is. It is the default for both tokens and application secrets when no hashing is configured.

- `transform_secret` returns the input unchanged.
- `allows_restoring_secrets?` returns `true` -- the plaintext can always be read back.
- Also serves as a fallback strategy when upgrading from plaintext to hashed storage.

### Sha256Hash

`Doorkeeper::SecretStoring::Sha256Hash` stores a SHA-256 hex digest of the secret. It is the default hashing strategy when you call `hash_token_secrets` or `hash_application_secrets` without a `using:` argument.

```ruby
hash_token_secrets
# Equivalent to:
# hash_token_secrets using: '::Doorkeeper::SecretStoring::Sha256Hash'
```

- `transform_secret` returns `::Digest::SHA256.hexdigest(plain_secret)`.
- `allows_restoring_secrets?` returns `false` -- the original secret cannot be recovered from the hash.
- Compatible with both token and application secrets.

### BCrypt (application secrets only)

`Doorkeeper::SecretStoring::BCrypt` stores application secrets as BCrypt password hashes. It is **only valid for application secrets** -- attempting to use it for token secrets raises an `ArgumentError`.

```ruby
# Gemfile
gem 'bcrypt', require: false

# config/initializers/doorkeeper.rb
hash_application_secrets using: '::Doorkeeper::SecretStoring::BCrypt'
```

- `transform_secret` returns `::BCrypt::Password.create(plain_secret.to_s)`.
- `secret_matches?` compares input against the stored BCrypt hash using `BCrypt::Password#==`.
- `allows_restoring_secrets?` returns `false`.
- `validate_for` enforces that the strategy is only used on `:application` models and that the `bcrypt` gem is loaded.

BCrypt is unsuitable for **access tokens** because tokens must be looked up by their plaintext value (e.g., during token introspection or when `reuse_access_token` is enabled). BCrypt's per-hash salt makes deterministic lookup impossible.


## Rotation best practices

Rotating secrets depends on what you are rotating and whether hashing is enabled.

**Application secrets (client_secret):**

1. Regenerate the secret on the `Doorkeeper::Application` record (e.g., via the admin panel or a rake task that calls `application.renew_secret`).
2. Distribute the new secret to all clients that authenticate with this application.
3. The old secret is immediately invalidated. If you use `fallback: :plain` and had plaintext secrets, the old plaintext value will still
4. be found until the record is updated with the new hashed value.

**Access tokens:**

1. Revoke existing tokens (e.g., via `Doorkeeper::AccessToken.revoke_all_for(application_id, resource_owner)` or individual `token.revoke`).
2. Issue new tokens through the normal OAuth flow. New tokens will be stored using the currently configured strategy.
3. If you recently enabled `hash_token_secrets`, old plaintext tokens remain valid via `fallback: :plain` and are upgraded on first use. To force rotation, revoke them and let clients re-authenticate.

**Token rotation with hashing enabled:**

Once `hash_token_secrets` is active, you cannot retrieve the plaintext of a previously issued token (the strategy returns `allows_restoring_secrets? == false`). The plaintext is only available once in `token.plaintext_token` immediately after creation. Design your token delivery flow accordingly -- show the token to the user at creation time and never again.


## Compatibility notes

### hash_token_secrets and reuse_access_token

`hash_token_secrets` is **incompatible** with `reuse_access_token`. The `reuse_access_token` feature works by looking up
an existing valid token for the same client/resource-owner combination and returning it instead of creating a new one.
This lookup requires the plaintext token, which a hashing strategy cannot provide.

When both are configured, Doorkeeper detects the conflict during validation (`validate_reuse_access_token_value` in
`Doorkeeper::Config::Validations`) and **automatically disables `reuse_access_token`** with a warning:

```
[DOORKEEPER] You have configured both reuse_access_token AND 'Sha256Hash'
strategy which cannot restore tokens. This combination is unsupported.
reuse_access_token will be disabled
```

The same applies to `token_reuse_limit` -- without `reuse_access_token`, it has no effect.

### Enabling hashing on an existing database

When you enable `hash_token_secrets` or `hash_application_secrets` on a database that already contains plaintext secrets, add `fallback: :plain` to keep those records valid:

```ruby
hash_token_secrets fallback: :plain
hash_application_secrets fallback: :plain
```

Without the fallback, Doorkeeper will look up secrets using only the SHA-256 digest of the input, and existing plaintext records will not match. With the fallback, the lookup first tries the current strategy, then falls back to `Plain` -- and if a match is found via the fallback, the record is transparently **upgraded** to the current strategy (`upgrade_fallback_value` in `Doorkeeper::Models::SecretStorable`).

New secrets created after enabling hashing are always stored with the current strategy. The fallback only affects existing records.

### Cross-references

See [Other Configurations](../configuration/other-configurations.md) for related settings including `reuse_access_token`, `token_reuse_limit`, and token generator options.

