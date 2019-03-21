---
description: How to use `skip_authorization` configuration option
---

# Skip Authorization

Users using the Authorization Code Flow will see this screen after logging in:

![](../.gitbook/assets/oauth-authorization-required-2019-03-21-20-22-14.png)

Under some circumstances, you might want to let users skip the screen above and have [applications](../concepts/application.md) auto-approved \(for example, when dealing with a trusted application\).

This is possible via a configuration option `skip_authorization` which takes either `true` or `false`:

{% code-tabs %}
{% code-tabs-item title="config/initializers/doorkeeper.rb" %}
```ruby
skip_authorization do
  true
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

If you need more control over which application or user can skip the authorization, the resource owner and client will be available as block arguments:

{% code-tabs %}
{% code-tabs-item title="config/initializers/doorkeeper.rb" %}
```ruby
skip_authorization do |resource_owner, client|
  client.superapp? || resource_owner.admin?
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

