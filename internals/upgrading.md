# Upgrading

If you want to upgrade Doorkeeper to a new version, check out the
[upgrading guides](https://github.com/doorkeeper-gem/doorkeeper/wiki/Migration-from-old-versions)
and take a look at the [CHANGELOG](https://github.com/doorkeeper-gem/doorkeeper/blob/master/CHANGELOG.md).

- **Per-application grant flow customization** ([#1286]). Override `allow_grant_flow_for_client?`
  on the Application model to control which grant flows each client can use.

- **Batch token lookup** ([#1270]). `reuse_access_token` now finds matching tokens in batches
  (configurable via `matching_token_for` lookup size). This improves performance for large
  token tables.

- **Client credentials token revocation** ([#1271]). 5.2 revokes the previous access token
  when a client requests new client-credentials tokens by default.
  **Note:** this default was reverted in 5.3. See the 5.3 section below.

- **`client_via_uid` removed** ([#1296]). `Doorkeeper::Server#client_via_uid` was removed;
  `client_id` is now part of strong parameters.

## Breaking changes in 5.3

- **Optional client credentials revocation** ([#1318]). Token revocation for client credentials
  grants is now **off by default** (reversing the 5.2 default). Re-enable with
  `revoke_previous_client_credentials_token` in the initializer if you relied on 5.2 behavior.

- **Custom model classes** ([#1345]). Doorkeeper models can now use custom classes via
  `application_class`, `access_token_class`, etc. Extracted AR mixins are available in
  `Doorkeeper::Orm::ActiveRecord::Mixins`.

- **CVE-2020-10187 backport** ([#1371] in 5.3.2). `Doorkeeper::Application#as_json` was
  restricted to a minimal set of serialized attributes, fixing an information disclosure
  vulnerability. Custom serialization on `/oauth/applications.json` must be reimplemented
  via `#as_json` overrides.

## Breaking changes in 5.4

- **Polymorphic resource owner** ([#1355]). Access Tokens and Grants can now belong to
  polymorphic resource owners via `use_polymorphic_resource_owner`. Doorkeeper passes
  resource owner **instances** (not just IDs) — review custom patches and extensions.

- **`authorize_resource_owner_for_client`** ([#1354]). A new hook to control authorization
  on a per-application basis before granting access.

- **CVE-2020-10187 fix (original)** ([#1371]). The `#as_json` serialization restriction
  was developed here and backported to 5.0–5.3.

- **RFC 7009 revocation** ([#1370]). Token revocation now requires `client_id` (public
  clients) and `client_secret` (private clients), per RFC 7009.

- **`active_record_options` deprecated** ([#1358]). Replaced by direct config options;
  removed in 5.6.3.

## Notable changes in 5.5–5.9

### 5.5

- **Custom grant flows** ([#1418]). Register custom OAuth grant types via
  `grant_flows` in the initializer.
- **Form-post response mode** ([#1438]). Support for `response_mode=form_post`.
- **Client authentication for password grant** ([#1420]). Resource Owner Password
  Grant now requires client authentication per RFC. Opt out with
  `skip_client_authentication_for_password_grant` (discouraged).
- **API-mode controllers** ([#1473]). `ApplicationsController` and
  `AuthorizedApplicationsController` are now available in API mode. Skip with
  `skip_controllers` if not needed.

### 5.6

- **Dynamic scopes** ([#1739] in 5.8.0, continued in 5.6.x). Wildcard and
  pattern-matched scopes.
- **Custom token attributes** ([#1602]). Store arbitrary data on access tokens
  and grants via `custom_access_token_attributes`.
- **`matching_token_for` API change** ([#1542]). Now returns only **active**
  tokens (not just non-revoked). This affects `reuse_access_token` behavior.

### 5.7

- **`force_pkce`** ([#1705]). Require non-confidential clients to use PKCE
  when requesting tokens via authorization code.

### 5.8

- **`pkce_code_challenge_methods`** ([#1735]). Configure allowed PKCE challenge
  methods (e.g. `S256`, `plain`).
- **Public application null secret** ([#1727]). Public applications can have a
  `nil` secret value.

### 5.9

- **Read-replica support** ([#1791]). `enable_multiple_database_roles` for
  automatic role switching in multi-database setups.
- **Idempotent revocation** ([#1778]). Revoking an already-revoked token no
  longer raises an error.
- **i18n view punctuation extraction (5.9.1)** ([#1784]). **Breaking for apps
  with custom views:** colons were removed from ERB templates and moved into
  i18n translation strings. If you override `authorizations/new`,
  `authorizations/show`, or `applications/show`, update your views and
  translations (or upgrade the [doorkeeper-i18n](https://github.com/doorkeeper-gem/doorkeeper-i18n) gem).
- **RFC 7009 public client fix (5.9.1)** ([#1806]). Fixed a token revocation
  bypass for public clients.
- **Client auth rejection (5.9.5)** ([#1901]). Clients sending credentials via
  more than one method are rejected with `invalid_request`.

## Migration checklist

1. **Generate and run migrations.**
   ```bash
   rails generate doorkeeper:migration
   rails db:migrate
   ```
   Depending on the version gap, additional generators may be needed:
   `doorkeeper:confidential_applications`, `doorkeeper:pkce`,
   `doorkeeper:enable_polymorphic_resource_owner`,
   `doorkeeper:previous_refresh_token`, etc.

2. **Review the initializer.** Compare your
   `config/initializers/doorkeeper.rb` against the latest template. Pay
   attention to removed or renamed options (e.g. `enable_pkce_without_secret`,
   `active_record_options`).

3. **Update custom views.** If you override any Doorkeeper views, review them
   against the current templates — especially for the 5.9.1 i18n punctuation
   change (colons moved from ERB to locale strings).

4. **Test token formats.** After upgrading from 5.0 to 5.1+, verify that
   existing hex tokens still validate. If you customized token generation,
   confirm `default_generator_method` is set appropriately.

5. **Audit client applications.** Ensure all applications have the
   `confidential` flag set correctly. Public clients (native/mobile apps,
   SPAs) should have `confidential: false`.

6. **Review grant flow and scope configuration.** Check that
   `scopes_by_grant_type`, `grant_flows`, and per-application
   `allow_grant_flow_for_client?` are configured as expected for your use
   cases.

## Deprecation notes

- `active_record_options` was deprecated in 5.4 and removed in 5.6.3.
- `enable_pkce_without_secret` was removed in 5.0.
- `Doorkeeper.configured?`, `Doorkeeper.database_installed?`, and
  `Doorkeeper.installed?` were removed in 5.0.

## Additional resources

- [Migration from old versions (wiki)](https://github.com/doorkeeper-gem/doorkeeper/wiki/Migration-from-old-versions)
- [CHANGELOG](https://github.com/doorkeeper-gem/doorkeeper/blob/master/CHANGELOG.md)
