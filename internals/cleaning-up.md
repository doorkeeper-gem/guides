# Cleaning Up

By default Doorkeeper is retaining expired and revoked access tokens and grants. This allows to keep an audit log of those records, but it also leads to the corresponding tables to grow large over the lifetime of your application.

If you are concerned about those tables growing too large, you can regularly run the following rake task to remove stale entries from the database:

```text
rake doorkeeper:db:cleanup
```

Note that this will remove tokens that are expired according to the configured TTL in `Doorkeeper.configuration.access_token_expires_in`. The specific `expires_in` value of each access token **is not considered**. The same is true for access grants.

