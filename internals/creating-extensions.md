# Creating extensions

In some cases you may want to have some functionality that extends Doorkeeper, but it's not something
that is required by the OAuth RFC and must be implemented in gem itself. In such cases you may create
an extension for Doorkeeper gem.

You can reuse Doorkeeper internals for your own purposes.

## Extension configuration

You can reuse Doorkeeper options DSL to define your own configurations.

```ruby
class YourExtension
  def self.configure(&block)
    @config = Config::Builder.new(Config.new, &block).build
  end

  def self.configuration
    @config || (raise Errors::MissingConfiguration)
  end

  class Config
    class Builder < Doorkeeper::Config::AbstractBuilder
      # your custom config methods
      def enforce_something
        @config.instance_variable_set(:@enforce_something, true)
      end
    end

    def enforce_something?
      if defined?(@enforce_something)
        @enforce_something
      else
        false
      end
    end
 
    # In case you defined some other builder class
    def self.builder_class
      Config::Builder
    end

    extend Doorkeeper::Config::Option

    # your custom options
    option :expiration_time, default: 1.hour
    option :skip_something, default: true
  end
end
```

Take a look at Doorkeeper config file to get more examples of options DSL and it's arguments.

## Examples

Examples of extensions in the wild:

* [doorkeeper-openid_connect](https://github.com/doorkeeper-gem/doorkeeper-openid_connect/tree/master/lib/doorkeeper)
* [doorkeeper-sequel](https://github.com/nbulaj/doorkeeper-sequel)
* [doorkeeper-grants_assertion](https://github.com/doorkeeper-gem/doorkeeper-grants_assertion)
