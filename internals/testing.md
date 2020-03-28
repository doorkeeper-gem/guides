# Testing

You can use Doorkeeper models in your application test suite. Note that starting from
Doorkeeper 4.3.0 it uses [ActiveSupport lazy loading hooks](http://api.rubyonrails.org/classes/ActiveSupport/LazyLoadHooks.html)
to load models. There are [known issue](https://github.com/doorkeeper-gem/doorkeeper/issues/1043)
with the `factory_bot_rails` gem (it executes factories building before `ActiveRecord::Base` is
initialized using hooks in gem railtie, so you can catch a `uninitialized constant` error).
It is recommended to use pure `factory_bot` gem to solve this problem.
