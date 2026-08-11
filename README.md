# Doorkeeper Guides

Doorkeeper is an OAuth 2.0 provider built in Ruby. It integrates with Ruby on Rails and Grape frameworks.

## Installation

The installation process depends on the framework you're using. The first step is to add `doorkeeper` to your `Gemfile`:

{% code-tabs %}
{% code-tabs-item title="# Gemfile" %}
```ruby
gem 'doorkeeper'
```
{% endcode-tabs-item %}
{% endcode-tabs %}

And run `bundle install`. After this, make sure to follow the guide related to the framework you're using below.

### Ruby on Rails

Doorkeeper follows Rails maintenance policy and supports only supported versions of the framework.
Currently we support Ruby on Rails >= 5.0. See the guide [here](ruby-on-rails/getting-started.md).

### Grape

Guide for integration with Grape framework can be found [here](grape/grape.md).

### ORMs

Doorkeeper supports `ActiveRecord` by default, but can be configured to work with the following ORMs:

| ORM | Support via |
| :--- | :--- |
| Active Record | by default |
| MongoDB | [doorkeeper-gem/doorkeeper-mongodb](https://github.com/doorkeeper-gem/doorkeeper-mongodb) |
| Sequel | [nbulaj/doorkeeper-sequel](https://github.com/nbulaj/doorkeeper-sequel) |
| Couchbase | [acaprojects/doorkeeper-couchbase](https://github.com/acaprojects/doorkeeper-couchbase) |
| RethinkDB | [aca-labs/doorkeeper-rethinkdb](https://github.com/aca-labs/doorkeeper-rethinkdb) |

### Database maintenance

Doorkeeper does **not** automatically remove expired or revoked tokens and grants. The `oauth_access_tokens` and `oauth_access_grants` tables grow indefinitely and can reach millions of rows if left unmanaged.

Prune them periodically with the bundled rake task:

```bash
bundle exec rake doorkeeper:db:cleanup
```

This deletes expired and revoked access tokens and grants. See the [Rake tasks guide](https://doorkeeper.gitbook.io/guides/internals/rake) for details.

### Extensions

Extensions that are not included by default and can be installed separately.

Please note that some of the extensions are not maintained by the Doorkeeper team, so please check the repository for the latest updates and issues.

|  | Link |
| :--- | :--- |
| OpenID Connect extension | [doorkeeper-gem/doorkeeper-openid\_connect](https://github.com/doorkeeper-gem/doorkeeper-openid_connect) |
| JWT Token support | [doorkeeper-gem/doorkeeper-jwt](https://github.com/doorkeeper-gem/doorkeeper-jwt) |
| Assertion grant extension | [doorkeeper-gem/doorkeeper-grants\_assertion](https://github.com/doorkeeper-gem/doorkeeper-grants_assertion) |
| I18n translations | [doorkeeper-gem/doorkeeper-i18n](https://github.com/doorkeeper-gem/doorkeeper-i18n) |
| CIBA (Client Initiated Backchannel Authentication) | [doorkeeper-gem/doorkeeper-ciba]([https://github.com/doorkeeper-gem/doorkeeper-ciba](https://github.com/autoseg/doorkeeper-ciba)) |
| Device Authorization Grant | [doorkeeper-gem/doorkeeper-device\_authorization\_grant]([https://github.com/doorkeeper-gem/doorkeeper-device_authorization_grant](https://github.com/exop-group/doorkeeper-device_authorization_grant)) |

### Example Applications

These applications show how Doorkeeper works and how to integrate with it. Start with the OAuth 2.0 server and use the clients to connect with the server.

| Application | Link |
| :--- | :--- |
| OAuth 2.0 Server with Doorkeeper | [doorkeeper-gem/doorkeeper-provider-app](https://github.com/doorkeeper-gem/doorkeeper-provider-app) |
| Sinatra Client connected to Provider App | [doorkeeper-gem/doorkeeper-sinatra-client](https://github.com/doorkeeper-gem/doorkeeper-sinatra-client) |
| Devise + Omniauth Client | [doorkeeper-gem/doorkeeper-devise-client](https://github.com/doorkeeper-gem/doorkeeper-devise-client) |

### Tutorials

See [list of tutorials](https://github.com/doorkeeper-gem/doorkeeper/wiki#how-tos--tutorials) in order to learn how to use the gem or integrate it with other solutions / gems.

