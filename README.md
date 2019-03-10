# Doorkeeper Guides

Doorkeeper is an oAuth2 provider built in Ruby. It integrates with Ruby on Rails and Grape frameworks.

## Installation

Installation depends on the framework you're using. The first step is to add the following to your Gemfile:

```ruby
gem 'doorkeeper'
```

And run `bundle install`. After this, check out the guide related to the framework you're using.

### Ruby on Rails

Doorkeeper currently supports [Ruby on Rails 5](ruby-on-rails/getting-started.md).

### Grape

Guide for integration with Grape framework can be found [here](grape/grape.md).

### ORMs

| ORM | Support via |
| :--- | :--- |
| Active Record | by default |
| MongoDB | [https://github.com/doorkeeper-gem/doorkeeper-mongodb](https://github.com/doorkeeper-gem/doorkeeper-mongodb) |
| Sequel | [https://github.com/nbulaj/doorkeeper-sequel](https://github.com/nbulaj/doorkeeper-sequel) |
| Couchbase | [https://github.com/acaprojects/doorkeeper-couchbase](https://github.com/acaprojects/doorkeeper-couchbase) |



