# Database Design

## oauth\_applications

| Field | Purpose |
| :--- | :--- |
| **id** | Primary key, in case of using RDBMs |
| **name** | Application name |
| **uid** | Unique ID, used as [_client identifier_](https://tools.ietf.org/html/rfc6749#section-2.2)\_\_ |
| **secret** | Used together with `uid` for client authentication |
| **redirect\_uri** | Redirects the resource owner to this URI \([spec](https://tools.ietf.org/html/rfc6749#section-3.1.2)\) |
| **scopes** | Defines which [scopes](../configuration/scopes.md) the application uses |
| **confidential** | Indicates whether client public or private |



