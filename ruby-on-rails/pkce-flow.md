# PKCE Flow

If you want to enable [PKCE flow](https://tools.ietf.org/html/rfc7636) for mobile apps, you need to generate another migration:

```text
bundle exec rails generate doorkeeper:pkce
```

This step is optional and you will be able to add this later if necessary.

