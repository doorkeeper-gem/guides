# PKCE Flow

## Links to specification:

* [RFC 7636 - Proof Key for Code Exchange by OAuth Public Clients](https://tools.ietf.org/html/rfc7636)
* [PKCE for OAuth 2.0](https://oauth.net/2/pkce/)

## Reason for **PKCE**

On mobile apps common practice was: Usage of one hardcoded client_id and secret for all app installations on
any device. By design this kind of client cannot be confidential. Mobile apps can be decompiled.
Although there are some techniques to hide the secret and make it harder to reconstruct it — the secret is
never really safe.

### The attack vector **PKCE** secures for:

App A is developed by trusted source. It's redirect uri may be something like: `my-own-app://back-from-oauth`
It implements the standard authorization code grant. It registers to `my-own-app://back-from-oauth`

App B is developed by some harmful Hacker who decompiled App A and received client_id and secret. It registers
to my-own-app://back-from-oauth, too. Somehow this Hacker gets a user of App A to install App B, too.
When the user now starts App A and has to login again - after the redirect App B opens (current Android
Versions now ask which app has to be opened - not so for iOS and old Android versions still on the market),
gets the authorization code and can do a token request to make a valid access token out of it.

### How does PKCE prevent authorization code interception?

**PKCE (Proof Key for Code Exchange)** extends the authorization code grant to
protect against authorization code interception attacks.

Before starting the authorization flow, the client generates a
cryptographically random string called the `code_verifier`. This value must
contain between 43 and 128 characters from the following set: `A-Z`, `a-z`,
`0-9`, `-`, `.`, `_`, and `~`.

The client then derives a `code_challenge` from the `code_verifier`. Doorkeeper
supports two challenge methods:

- **`S256` (recommended):** The client computes the SHA-256 hash of the
  `code_verifier` and encodes the result using Base64url without padding.
- **`plain`:** The `code_challenge` is identical to the `code_verifier`. This
  method is supported for compatibility but should be avoided.

When the client redirects the user to `/oauth/authorize`, it includes
`code_challenge` and `code_challenge_method` alongside the standard
authorization request parameters, such as `client_id`, `redirect_uri`,
`response_type=code`, `scope`, and `state`.

After the user authorizes the request, Doorkeeper redirects the browser to the
client's `redirect_uri` with an authorization code.

The client then sends a POST request to `/oauth/token`, including the
authorization `code`, the original `code_verifier`, and the other required
token request parameters.

Doorkeeper applies the previously specified `code_challenge_method` to the
submitted `code_verifier` and compares the result with the `code_challenge`
associated with the authorization code. If the values do not match, Doorkeeper
rejects the token request with an `invalid_grant` error.

**This prevents authorization code interception attacks.** Even if a malicious
application (App B) intercepts the authorization code, it cannot exchange that
code for an access token without the original `code_verifier`, which is known
only to the legitimate client (App A).

## Enable PKCE in Doorkeeper

If you want to enable [PKCE flow](https://tools.ietf.org/html/rfc7636) for mobile apps, you need to
generate another migration:

```text
bundle exec rails generate doorkeeper:pkce
```

This step is optional and you will be able to add this later if necessary.

If you overrode the `doorkeeper/authorizations/new.html.erb` view, make sure that you have the `code_challenge` and `code_challenge_method` hidden form fields.
  
Also, in case your client is public (e.g. mobile app, single page app) note that the `Doorkeeper::Application` for the client should have `confidential: false`. Otherwise Doorkeeper will try to authenticate the client using the client_secret, which the client, being public, shouldn't know about.

## PKCE and refresh tokens

Discussion over PKCE flow and refresh tokens you can find here: https://github.com/doorkeeper-gem/doorkeeper/issues/1285

## Forcing PKCE

Starting from Doorkeeper 5.7.1, you can require PKCE for all non-confidential (public) clients
by using the `force_pkce` option. When enabled, any public client that uses the authorization code
grant flow must include a `code_challenge` and subsequently provide a valid `code_verifier` in the
token request. Confidential clients are not affected.

{% tabs %}
{% tab title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper.configure do
  force_pkce
end
```
{% endtab %}
{% endtabs %}

This is particularly useful when you know your application only serves public clients (e.g., mobile
apps, single-page applications) and want to ensure the authorization code flow is always protected
against code interception attacks.

## Code Challenge Methods

The `pkce_code_challenge_methods` option controls which code challenge transformation methods are
accepted. The default is `%w[plain S256]`, supporting both methods.

For better security, you can restrict Doorkeeper to accept only the `S256` method:

{% tabs %}
{% tab title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper.configure do
  pkce_code_challenge_methods %w[S256]
end
```
{% endtab %}
{% endtabs %}

Only `plain` and `S256` are valid values. Custom code challenge methods are not supported: if the
option contains any other value, Doorkeeper logs a warning at boot and resets the option to the
default `%w[plain S256]`, which silently re-enables `plain` even if you intended to disable it.

{% hint style="info" %}
**Security recommendation:** Prefer `S256` over `plain`. The `plain` method transmits the
`code_verifier` value directly as the `code_challenge`, which provides no cryptographic
protection against an attacker who can intercept the authorization request.
{% endhint %}
