# PKCE Flow

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

### How can **PKCE** prevent this?

**PKCE** modifies the standard authorization code grant. Before starting the authorization code grant, a
random string called `code_verifier` is generated using the characters: [A-Z], [a-z], [0-9], "-", ".", "_"
and "~", with a minimum length of 43 characters and a maximum length of 128 characters. Additionally to the
`code_verifier` a `code_challenge` is created. Doorkeeper supports two code_challenge_methods to generate
the `code_challenge`. With `code_challenge_method` "plain" the `code_challenge` is the same as the `code_verifier`.
With `code_challenge_method` "S256" the `code_challenge` is the SHA256 Hash value of the `code_verifier` url
safe base64 encoded (without trailing "=" compare to: https://tools.ietf.org/html/rfc7636#appendix-A)

Now when the mobile app redirects into the browser to https://<doorkeeper-app>/oauth/authorize, add two
parameters. Next to `client_id`, `redirect_uri`, `response_type=code`, `scope` or `state`, you add now
`code_challenge` and `code_challenge_method`. *Remember*: doorkeeper supports the two methods "plain" and
"S256". We do not recommend to use plain.

When your app opens after the redirect to the redirect_uri and the app got the `authorization_code`,
add another parameter to your token request. Next to `code`, `client_id` or `grant_type`, you add the
`code_verifier` to your /oauth/token POST request. Doorkeeper will now use the already known
`code_challenge_method` to create its own `code_challenge` from `code_verifier` — and compare it to
the already stored `code_challenge` from `/oauth/authorize` request. If they do not match, you get an
error message to have done an `invalid_request`.

If now App B receives the authentication code, it cannot request a token for it, since it cannot know the nonce that was created only for this one request.

## Enable PKCE in Doorkeeper

If you want to enable [PKCE flow](https://tools.ietf.org/html/rfc7636) for mobile apps, you need to
generate another migration:

```text
bundle exec rails generate doorkeeper:pkce
```

This step is optional and you will be able to add this later if necessary.

