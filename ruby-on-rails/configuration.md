# Configuration

Before you're able to use Doorkeeper, you need to configure how [resource owners](../concepts/resource-owner.md) \(users\) can be authenticated and who can manage such [applications](../concepts/application.md).

## Resource Owner Authentication

This configuration should do two things:

1. Return the user is currently authenticated
2. Redirect the user to the authentication page

If you're using [devise](https://github.com/plataformatec/devise), one option is to write the following:

{% code-tabs %}
{% code-tabs-item title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper.configure do
  resource_owner_authenticator do
    current_user || warden.authenticate!(scope: :user)
  end
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

The block above runs in the context of your application so you have access to your models, session
and routes helpers. However, it is **not** run in the context of the `ApplicationController` which
means that it doesn't have access to the methods defined over there.

You may want to check other ways of authentication [here](https://github.com/doorkeeper-gem/doorkeeper/wiki/Authenticating-using-Clearance-or-DIY).

## Application Management Authentication

By default, the applications list in `/oauth/applications` is unavailable. To let users see and
manage **all applications**, you should configure `admin_authenticator` block:

{% code-tabs %}
{% code-tabs-item title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper.configure do
  admin_authenticator do |_routes|
    current_user || warden.authenticate!(scope: :user)
  end
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

The block follows the same rules as `resource_owner_authenticator` block.

{% hint style="danger" %}
**Note:** the application list is just a scaffold. It's highly recommended to either customize the
controller used by the list or skip the controller all together. For more information see the page
[in the wiki](https://github.com/doorkeeper-gem/doorkeeper/wiki/Customizing-routes).
{% endhint %}
