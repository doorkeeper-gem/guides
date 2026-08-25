# Configuration

Before you're able to use Doorkeeper, you need to configure how resource owners \(users\) can be authenticated and who can manage such applications.

## Resource Owner Authentication

This configuration should do two things:

1. Return the user is currently authenticated
2. Redirect the user to the authentication page

If you're using [devise](https://github.com/plataformatec/devise), one option is to write the following:

{% tabs %}
{% tab title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper.configure do
  resource_owner_authenticator do
    current_user || warden.authenticate!(scope: :user)
  end
end
```
{% endtab %}
{% endtabs %}

The block above runs in the context of your application so you have access to your models, session
and routes helpers. However, it is **not** run in the context of the `ApplicationController` which
means that it doesn't have access to the methods defined over there.

You may want to check other ways of authentication [here](https://github.com/doorkeeper-gem/doorkeeper/wiki/Authenticating-using-Clearance-or-DIY).

## Application Management Authentication

By default, the applications list in `/oauth/applications` is unavailable. To let users see and
manage **all applications**, you should configure `admin_authenticator` block:

{% tabs %}
{% tab title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper.configure do
  admin_authenticator do |_routes|
    current_user || warden.authenticate!(scope: :user)
  end
end
```
{% endtab %}
{% endtabs %}

The block follows the same rules as `resource_owner_authenticator` block.

{% hint style="danger" %}
**Note:** the application list is just a scaffold. It's highly recommended to either customize the
controller used by the list or skip the controller all together. For more information see the page
[in the wiki](https://github.com/doorkeeper-gem/doorkeeper/wiki/Customizing-routes).
{% endhint %}

## Configuration overview

The `Doorkeeper.configure` block in `config/initializers/doorkeeper.rb` is the central place for
all Doorkeeper settings. Below is a high-level summary of the configuration categories available.
For detailed information on each topic, follow the cross-links.

- **Token settings** — expiration times, token generation, reuse, and revocation policies.
  See [Other Configurations](../configuration/other-configurations.md) and
  [Token Revocation](../configuration/token-revocation.md).
- **Grant flows** — enabled grant types (authorization code, client credentials, implicit, password)
  and per-client grant flow restrictions.
  See [Other Configurations](../configuration/other-configurations.md).
- **Scopes** — default and optional scopes, scope enforcement, and scopes by grant type.
  See [Scopes](../configuration/scopes.md).
- **Redirect URIs** — SSL enforcement, forbidden URI schemes, and blank redirect URI handling.
  See [Other Configurations](../configuration/other-configurations.md).
- **Application ownership** — opt-in application owner association with optional confirmation.
  See [Models](../configuration/models.md).
- **Token introspection** — custom introspection response fields and access control.
  See [Token Introspection](../configuration/token-introspection.md).
- **ORM and models** — ORM selection (`:active_record`), custom model classes for
  `AccessToken`, `AccessGrant`, and `Application`.
  See [Models](../configuration/models.md).
- **Controllers** — base controller class, API-only mode, and custom controller overrides.
  See [Other Configurations](../configuration/other-configurations.md).
- **Security and hashing** — token and application secret hashing (SHA256, BCrypt), PKCE
  enforcement, and client authentication methods.
  See [Token and Application Secrets](../security/token-and-application-secrets.md).
