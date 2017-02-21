
# Security

We use [`OAuth2`](https://oauth.net/2/) to authenticate users in the API and
[`Symfony ACLs`](http://symfony.com/doc/2.7/security/acl.html) to handle authorizations.

## Authentication

### Creating an OAuth client

The first operation you will need to authorize third-party applications to use the API is to create a client id.
A client id identifies a client application.

To increase security, the OAuth protocol specifies that a client id comes with a secret. They are always generated, used and revoked together.

To create a new pair of client id / secret, use the following command:

```bash
php bin/console pim:oauth-server:create-client \
    --grant-type="password" \
    --grant-type="refresh_token"
```

You will get something like:

```bash
A new client has been added:
client_id: 4gm4rnoizp8gskgkk080ssoo80040g44ksowwgw844k44sc00s
secret: 5dyvo1z6y34so4ogkgksw88ookoows00cgoc488kcs8wk4c40s
```

These keys must be transmitted by the administrator to any third-party application wanting to use the API.

Client ids are not enough to access the API, it's the role of tokens. Client ids allow two things:
asking a token for a specific user, and refresh an expired token.

:::info
In the case you want to give a time-limited access, you can create the client id without specifying the
`--grant-type="refresh_token"` option. This way, the client application can only access the API until the first token
expires, then will be unable to refresh it.
:::

### Revoking a client

To revoke a client id use the following command:

```shell
php bin/console pim:oauth-server:revoke-client 4gm4rnoizp8gskgkk080ssoo80040g44ksowwgw844k44sc00s
```

:::warning
Client ids can be revoked at any moment by the administrator. In this case, all tokens created with the revoked client id will be invalidated and the application using this client id will be unable to ask a new one.
:::

### Listing all client

To get the list of all existing client ids use the following command:

```shell
php bin/console pim:oauth-server:list-clients
```

You will get this answer.
```bash
+----------------------------------------------------+----------------------------------------------------+
| Client id                                          | Secret                                             |
+====================================================+====================================================+
| 3e2iqilq2ygwk0ccgogkcwco8oosckkkk4gkoc0k4s8s044wss | 44ectenmudus8g88w4wkws84044ckw0k4w4kg0sokoss84oko8 |
+----------------------------------------------------+----------------------------------------------------+
```

### Getting a token

A client id has been created and provided to the client application. The last information needed to use the API is the token.

To get a valid token, the client application must send the following request:

```bash
curl -X POST 
    -H "Content-Type: application/json"
    -H "Authorization: Basic MV8yMHFkY2hxN3drZTh3MDRna2NnMDBnOGN3a3NzczA0a3cwMGcwa2d3d280a2c4azg4ODo1MHN5a3R6c29xZ3cwYzRnY280b2djOHdrazBvOHN3Z2tzYzR3MHNrZ3dzb2d3bzBnbw=="
    -d '{
        "grant_type": "password",
        "username": "admin",
        "password": "admin"
    }' 
    "http://test-dev-feature-9.akeneo.com/api/oauth/v1/token"
```

```http
POST /api/oauth/v1/token HTTP/1.1
Host: test-dev-feature-9.akeneo.com
Content-Type: application/json
Authorization: Basic MV8yMHFkY2hxN3drZTh3MDRna2NnMDBnOGN3a3NzczA0a3cwMGcwa2d3d280a2c4azg4ODo1MHN5a3R6c29xZ3cwYzRnY280b2djOHdrazBvOHN3Z2tzYzR3MHNrZ3dzb2d3bzBnbw==

{
"grant_type": "password",
"username": "admin",
"password": "admin"
}
```

**Response example**
```json
{
    "access_token": "NzFiYTM4ZTEwMjcwZTcyZWIzZTA0NmY3NjE3MTIyMjM1Y2NlMmNlNWEyMTAzY2UzYmY0YWIxYmUzNTkyMDcyNQ",
    "expires_in": 3600,
    "token_type": "bearer",
    "scope": null,
    "refresh_token": "MDk2ZmIwODBkYmE3YjNjZWQ4ZTk2NTk2N2JmNjkyZDQ4NzA3YzhiZDQzMjJjODI5MmQ4ZmYxZjlkZmU1ZDNkMQ"
}
```

- `access_token` is the information that must be included in every requests to the API for the client application to be authorized (see the example below).
- `expires_in` is the token lifespan (in seconds).
- `refresh_token` is a special token used only to refresh your `access_token` after it expired.

.. image:: ./img/authentication.png

### Accessing a resource

```bash
curl https://demo.akeneo.com/api/rest/v1/categories
    -H "Authorization: Bearer NzFiYTM4ZTEwMjcwZTcyZWIzZTA0NmY3NjE3MTIyMjM1Y2NlMmNlNWEyMTAzY2UzYmY0YWIxYmUzNTkyMDcyNQ"
```

### Refreshing an expired token

For security reasons, access tokens have a certain lifespan. A client application using an expired token will be unable
to request the API.

After a token has expired, the client application can ask a new token by calling:

```bash
GET https://demo.akeneo.com/api/oauth/v1/token
    -d "client_id"=4gm4rnoizp8gskgkk080ssoo80040g44ksowwgw844k44sc00s \
    -d "client_secret"=5dyvo1z6y34so4ogkgksw88ookoows00cgoc488kcs8wk4c40s \
    -d "grant_type"=refresh_token \
    -d "refresh_token"=MDk2ZmIwODBkYmE3YjNjZWQ4ZTk2NTk2N2JmNjkyZDQ4NzA3YzhiZDQzMjJjODI5MmQ4ZmYxZjlkZmU1ZDNkMQ
```

The response will be the same than when asking for the initial token except it will contain a new `access_token`
and `refresh_token`.

.. image:: ./img/refreshing_access_token.png

## Authorization

Like when using the PIM through the UI, ACLs are here to define what a user can and cannot do.
In the role form, a `Web API permissions` tab includes several ACLs for the API.

The first one `Overall Web API access` means that if enable for a role, each user depending on that role will have access to the Web API.

You can tune more finely this permission by restricting or allowing the access to the entities of the catalog structure (categories, families, attributes and attribute options). Here is the complete list of the availables ACLs :
- List attributes
- List attribute options
- List categories
- List families
- Create and update attributes
- Create and update attribute options
- Create and update categories
- Create and update families

.. image:: ./img/acl.png

:::warning
As accessing the API grants higher privileges than when using the UI, we strongly recommend creating one or more dedicated users with specific roles for the web API.
:::