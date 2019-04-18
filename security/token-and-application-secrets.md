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
To enable it, simply add it to your Gemfile: 

{% code-tabs %}
{% code-tabs-item title="Gemfile" %}
```ruby
gem 'doorkeeper'
gem 'bcrypt', require: false
```
{% endcode-tabs-item %}
{% endcode-tabs %}

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

