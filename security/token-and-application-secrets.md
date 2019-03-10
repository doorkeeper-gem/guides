# Token and Application Secrets

Doorkeeper can optionally Hash access, refresh tokens, and application secrets before persisting them. Since integrations may have decided on showing application secrets later on, both options can be enabled individually.

To enable hashing of access and refresh tokens, uncomment the initializer line `hash_token_secrets`. Upon issueing a new token, the plain token value will be available for presenting to the user in `token.plaintext_token`. Later comparisons will only be performed on the hashed token and the plain token can no longer be retrieved.

**Fallback to plain token lookup**

When you are upgrading doorkeeper and have existing access or refresh tokens in your database that you want to keep, enabling `hash_token_secrets` would implicitly invalidate all plaintext tokens since the plain tokens would no longer be found.

To enable plain values to be found and upgraded to a hashed value whenever it is accessed, you may enable the additional option `fallback_to_plain_secrets`. This is only needed for _existing_ doorkeeper installations with plain tokens present.

**Incompatibility with reuse\_access\_token**

Enabling `hash_token_secrets` is incompatible with the option `reuse_access_token` since plain values can no longer be retrieved. If you enabled both, the latter will be disabled with a warning.

To enable hashing application secrets \(`client_secret`\), uncomment the initializer line `hash_application_secrets`. This will result in the `secret` value of the application being available during the request that created in as `application.plaintext_secret`. In this request, you need to ensure the user noted the secret since you will no longer be able to show it afterwards.

**Using BCrypt gem for application secrets**

The hashing implementation for `Doorkeeper::Application` will try to load and use the `bcrypt` gem for hashing the secret. If the gem can not be loaded, it is falling back to a `SHA256` digest.

If you want to use bcrypt, simply add it to your Gemfile.

{% code-tabs %}
{% code-tabs-item title="Gemfile" %}
```ruby
gem 'doorkeeper'
gem 'bcrypt', require: false
```
{% endcode-tabs-item %}
{% endcode-tabs %}

