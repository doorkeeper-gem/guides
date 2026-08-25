# Resource Indicators

Doorkeeper supports [Resource Indicators for OAuth 2.0 (RFC 8707)](https://datatracker.ietf.org/doc/html/rfc8707) (**from version 6.0.0**),
allowing clients to signal which protected resource(s) they intend to access. Tokens are then audience-restricted to those resources.

## Setup

1. Run the generator to add the required `resource` column:

```text
bundle exec rails generate doorkeeper:resource_indicators
bundle exec rails db:migrate
```

2. Configure a validator in your initializer:

{% tabs %}
{% tab title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper.configure do
  resource_indicator_validator ->(resource_indicators, client) {
    allowed = %w[https://api.example.com/ https://calendar.example.com/]
    resource_indicators.all? { |r| allowed.include?(r) }
  }
end
```
{% endtab %}
{% endtabs %}

The callable receives an array of resource URIs and the OAuth client. Return `true` to accept or `false` to reject with `invalid_target`.

## Behavior

- Resource URIs must be absolute and must not contain a fragment component.
- Resource indicators are stored on grants and tokens.
- Token and refresh requests enforce subset restrictions against the original grant.
- Token introspection responses include `aud` when resource indicators are present.
- Grants issued with resource indicators retain their audience restriction even if the validator is later removed from configuration.

## Multiple resources

RFC 8707 uses repeated query parameters (`?resource=…&resource=…`) for multiple values, but Rack collapses repeated keys to the last value. Clients must use the Rails bracket syntax for multiple resource indicators:

```text
?resource[]=https://api.example.com/&resource[]=https://calendar.example.com/
```

A single `resource=…` parameter works as-is.

## See also

- [Grant Flows](../ruby-on-rails/grant-flows.md)
- [Other Configurations](../configuration/other-configurations.md)
