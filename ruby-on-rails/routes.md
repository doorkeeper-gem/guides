# Routes

The installation script will also automatically add the Doorkeeper routes into your app:

{% code-tabs %}
{% code-tabs-item title="config/routes.rb" %}
```ruby
Rails.application.routes.draw do
  use_doorkeeper
  # your routes below
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

This will mount following routes:

```text
   native_oauth_authorization GET    /oauth/authorize/native(.:format)             doorkeeper/authorizations#show
          oauth_authorization GET    /oauth/authorize(.:format)                    doorkeeper/authorizations#new
                              DELETE /oauth/authorize(.:format)                    doorkeeper/authorizations#destroy
                              POST   /oauth/authorize(.:format)                    doorkeeper/authorizations#create
                  oauth_token POST   /oauth/token(.:format)                        doorkeeper/tokens#create
                 oauth_revoke POST   /oauth/revoke(.:format)                       doorkeeper/tokens#revoke
             oauth_introspect POST   /oauth/introspect(.:format)                   doorkeeper/tokens#introspect
           oauth_applications GET    /oauth/applications(.:format)                 doorkeeper/applications#index
                              POST   /oauth/applications(.:format)                 doorkeeper/applications#create
        new_oauth_application GET    /oauth/applications/new(.:format)             doorkeeper/applications#new
       edit_oauth_application GET    /oauth/applications/:id/edit(.:format)        doorkeeper/applications#edit
            oauth_application GET    /oauth/applications/:id(.:format)             doorkeeper/applications#show
                              PATCH  /oauth/applications/:id(.:format)             doorkeeper/applications#update
                              PUT    /oauth/applications/:id(.:format)             doorkeeper/applications#update
                              DELETE /oauth/applications/:id(.:format)             doorkeeper/applications#destroy
oauth_authorized_applications GET    /oauth/authorized_applications(.:format)      doorkeeper/authorized_applications#index
 oauth_authorized_application DELETE /oauth/authorized_applications/:id(.:format)  doorkeeper/authorized_applications#destroy
             oauth_token_info GET    /oauth/token/info(.:format)                   doorkeeper/token_info#show

```

At this point on the guide, we're not going to change the routes. For more information on how to customize them, check out [this page on the wiki](https://github.com/doorkeeper-gem/doorkeeper/wiki/Customizing-routes).



