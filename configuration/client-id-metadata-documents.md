# Client ID Metadata Documents

Doorkeeper supports [OAuth Client ID Metadata Documents](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-client-id-metadata-document) (**from version 6.0.0**) for public clients.
A client such as an MCP client or an agent identifies itself with the `https://` URL of a JSON document describing it, instead of registering with your server first.
Doorkeeper fetches the document, keeps what it says as an application row, and the authorization code flow runs as for a registered public client.

The specification is still an IETF draft, so the behavior may change with it.

## Setup

1. Run the generator to add the `client_id_metadata_materialized_at` column, which tells a row written from a document apart from a registered application (ActiveRecord only).
   The flow also needs the PKCE columns (see [PKCE Flow](../ruby-on-rails/pkce-flow.md)):

```text
bundle exec rails generate doorkeeper:client_id_metadata_documents
bundle exec rails db:migrate
```

2. Enable the option in your initializer, with the scopes such a client may be granted at most. `none` has to be among the `client_authentication` methods, which it is by default:

{% tabs %}
{% tab title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper.configure do
  use_client_id_metadata_documents scopes: %w[read]

  # Without this, any https URL can become a client.
  allow_client_id_metadata_document do |client_id|
    %w[claude.ai chatgpt.com].include?(URI(client_id).host)
  end
end
```
{% endtab %}
{% endtabs %}

`scopes` is a hard cap: a document client never gets more, whatever its document asks for. A document without a `scope` field gets all scopes in the cap; one asking only for scopes outside the cap is refused. Always set it, as anyone can publish a document.

`allow_client_id_metadata_document` decides which `client_id`s are accepted, before anything is fetched. A client it stops accepting is refused from then on, refresh included. Compare the host, not a prefix: `start_with?("https://claude.ai")` also matches `claude.ai.evil.com`.

The [authorization server metadata](../ruby-on-rails/grant-flows.md#authorization-server-metadata-rfc-8414) at `/.well-known/oauth-authorization-server` advertises `client_id_metadata_document_supported` once the option is on.

## Behavior

- A `client_id` is taken for a document URL when it is `https`, has a path other than `/`, is at most 255 characters long and carries no userinfo, fragment or dot segments. Any other `client_id` is looked up as a registered `uid`.
- The document must name its own URL as `client_id`, list at least one `redirect_uris` entry, and declare `token_endpoint_auth_method` as `none`; a document asking for a shared secret, or naming no method, is refused.
- Redirect URIs your server refuses are dropped, the rest kept; a document left with none is refused.
- These are public clients: they get authorization codes only, always with S256 PKCE, and [`skip_authorization`](skip-authorization.md) does not apply, so the user consents on every authorization.
- A failed fetch refuses the client, also one seen before (the draft forbids using a stale document): new authorizations fail until its host answers again, while issued tokens and refresh keep working. A changed document updates the client on the next fetch.
- A registered application whose `uid` is a URL is used as registered and never fetched.
- Turning the option off refuses the clients written from documents, but access tokens already issued stay valid until they expire.

## Native clients and loopback redirects

`force_ssl_in_redirect_uri` is on everywhere but development, so the loopback redirects of native clients ([RFC 8252 Section 7.3](https://datatracker.ietf.org/doc/html/rfc8252#section-7.3)) are dropped unless you let them through:

{% tabs %}
{% tab title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper.configure do
  force_ssl_in_redirect_uri { |uri| !%w[127.0.0.1 ::1 localhost].include?(uri.hostname) }
end
```
{% endtab %}
{% endtabs %}

## Consent screen

The name of an application written from a document is its host followed by the document's `client_name` (`example.com: Example App`), or the host alone when the document gives none.
The host is the only verified hint of who is asking, as anyone can write any `client_name`.
The default `authorizations/new` view shows the application's `name`, so an [overridden view](../ruby-on-rails/customizing-views.md) that does the same needs no change; one that shows something else should keep the host visible.

## Fetching and rate limiting

Fetches are bounded: https only, no redirects, a few seconds and kilobytes, and no private or special-use addresses.
Failed fetches are not cached, so rate limiting of the authorization endpoint is left to the host application.

With `validate_client_before_resource_owner_authentication`, documents are fetched and applications created before anyone signs in, so without an allow hook anyone can create them anonymously.

## See also

- [PKCE Flow](../ruby-on-rails/pkce-flow.md)
- [Grant Flows](../ruby-on-rails/grant-flows.md)
- [Customizing Views](../ruby-on-rails/customizing-views.md)
- [Other Configurations](other-configurations.md)
