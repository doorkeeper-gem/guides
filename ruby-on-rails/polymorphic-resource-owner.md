# Polymorphic Resource Owner

Enables polymorphic Resource Owner feature for Doorkeeper Access Tokens & Grants.
By default Doorkeeper set only `resource_owner_id` without knowing model name of the Resource Owner.

Feature available only starting from Doorkeeper 5.4.

### How to use it

Enable configuration option:

{% code-tabs %}
{% code-tabs-item title="config/initializers/doorkeeper.rb" %}
```ruby
Doorkeeper.configure do
  orm :active_record

  use_polymorphic_resource_owner
end
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Generate migration to add polymorphic columns to tables:

```bash
$ bundle exec rails g doorkeeper:enable_polymorphic_resource_owner
```

If you previously set foreign key for `:resource_owner_id` column - you need to drop it as now it
would be a polymorphic association. You can drop FK in the migration generated above.

Run the migration:

```bash
$ bundle exec rails db:migrate
```

Use the feature!

Now you can access resource owner by `doorkeeper_token.resource_owner` association.

:warning: :warning:  **[ATTENTION]**: If you enabled this option on an existing project — you need to
manually set polymorphic type column with the class of the Resource Owner. Otherwise your existing
users/clients could face issues and spam you logs with errors.

You will need to run something like this in all your environments — replace `'Admin'` with your previous resource owner type:

```ruby
Doorkeeper::AccessToken.update_all(resource_owner_type: 'Admin')
```

:warning: :warning:  **[ATTENTION]**: Also this feature could be a breaking change for those users
that patched Doorkeeper internals, specially models. You need to fix your patches to user `resource_owner`
instance instead of it's ID (like it was before).

## Known projects affected

* GitLab (https://gitlab.com/gitlab-org/gitlab-foss/blob/master/config/initializers/doorkeeper.rb#L129)
* Doorkeeper ORM extensions

Original PR: https://github.com/doorkeeper-gem/doorkeeper/pull/1355
