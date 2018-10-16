# Authorization / Authentication microservice

### Why 
- Get an opportunity to restrict users, so that they can't send requests to microservices without required credentials and permissions.
- Provide for developers the solution for cases when OpenMatchmaking should work with the dedicated authorization/authentication server, withot communicating with existing authorization/authentication servers.
- Necessary to store an information about users and let them to get (or refresh) an access token.
- Required a storage for microservices permissions, so that they can be assigned to each existing user

### Benefits of using
- Can be used as the 3rd party authorization/authentication service for restricting an access in a pair with [reverse proxy](https://github.com/OpenMatchmaking/documentation/blob/master/docs/components/reverse-proxy.md#reverse-proxy)
- A good choice for cases when Open Matchmaking project should work independently, without communicating with other services (not related to Open Matchmaking)

### General information
General principles of auth/auth microservice can be shown with the following picture:

<p align="center">
  <img src="https://github.com/OpenMatchmaking/documentation/blob/master/docs/images/microservice-auth-schema.png"/>
</p>

As you can see on the picture, the microservice is communicating with two different entities:
1) With reverse proxy, which is sending requests to the certain microservice (in this case they are to Auth/Auth microservice) with a content inside.
2) With internal parts of Open Matchmaking which are communicating via specially created message queues for auth/auth microservice. It's a hidden part, a **don't provide** any public APIs for endusers. Only for internal usage.

### Entity relationship diagram
<p align="center">
  <img src="https://github.com/OpenMatchmaking/documentation/blob/master/docs/images/microservice-auth-db.png"/>
</p>

**IMPORTANT**: The tables "Group", "Permission" and "Microservice" will be updated by the other part of microservice, that will be handling requests from inner microservices after their instantiating.

#### User table
| Field name | Field type | Constraints      | Description                         |
|------------|------------|------------------|-------------------------------------|
|id          | UUID       | PK               | A database unique object identifier |
|username    | VARCHAR    | UNIQUE, NOT NULL | A unique username                   |
|password    | VARCHAR    | NOT NULL         | User password hash                  |
|group       | Relation   |                  | Many-To-Many Relationship to Group  |

#### Group table
| Field name | Field type | Constraints      | Description                             |
|------------|------------|------------------|-----------------------------------------|
|id          | UUID       | PK               | A database unique object identifier     |
|name        | VARCHAR    | NOT NULL         | A permission group name                 |
|permissions | Relation   |                  | Many-To-Many Relationship to Permission |

**NOTE**: On the first start of this microservice, the Group table should contain the following groups without permissions if they aren't created yet: "Game client". 

#### Permission table
| Field name | Field type | Constraints      | Description                                                      |
|------------|------------|------------------|------------------------------------------------------------------|
|id          | UUID       | PK               | A database unique object identifier                              |
|codename    | VARCHAR    | NOT NULL         | A permission name in "{microservice}.{resource}.{action}" format |
|description | VARCHAR    |                  | Human-readable description about the permission                  |

#### Microservice table
| Field name | Field type | Constraints      | Description                                              |
|------------|------------|------------------|----------------------------------------------------------|
|id          | UUID       | PK               | A database unique object identifier                      |
|name        | VARCHAR    | UNIQUE, NOT NULL | A unique Microservice identifier                         |
|version     | VARCHAR    | NOT NULL         | Microservice version in "{major}.{minor}.{patch}" format |
|permissions | Relation   |                  | One-To-Many Relationship to Permission                   |

### API Endpoints
| Method | Endpoint               | Usage                                  | Returns |
|--------|------------------------|----------------------------------------|---------|
| GET    | /auth/api/health-check | Health check endpoint for API Gateways | "OK"    |

### RabbitMQ exchanges and queues
#### Exchanges
| Exchange name                               | Exchange type | Options                                        |
|---------------------------------------------|---------------|------------------------------------------------| 
| open-matchmaking.auth.token.new.direct      | direct        | durable=True, passive=False, auto_delete=False |
| open-matchmaking.auth.token.verify.direct   | direct        | durable=True, passive=False, auto_delete=False |
| open-matchmaking.auth.token.refresh.direct  | direct        | durable=True, passive=False, auto_delete=False |
| open-matchmaking.auth.users.retrieve.direct | direct        | durable=True, passive=False, auto_delete=False |
| open-matchmaking.auth.users.register.direct | direct        | durable=True, passive=False, auto_delete=False |

#### Queues
| Queue name                  | Queue options                                                   | Exchange name                               | Usage                                                             | Returns                                    |
|-----------------------------|-----------------------------------------------------------------|---------------------------------------------|-------------------------------------------------------------------|--------------------------------------------|
| auth.microservices.register | durable=True, passive=False, exclusive=False, auto_delete=False | open-matchmaking.direct                     | Register a new microservice with permissions                      | "OK" or a validation error                 |
| auth.token.new              | durable=True, passive=False, exclusive=False, auto_delete=False | open-matchmaking.auth.token.new.direct      | Get an access token (or an access token and refresh token)        | A new JSON Web Token or a validation error |
| auth.token.verify           | durable=True, passive=False, exclusive=False, auto_delete=False | open-matchmaking.auth.token.verify.direct   | Returns with whether or not a given access token is valid         | {"is_valid": true} or a validation error   |
| auth.token.refresh          | durable=True, passive=False, exclusive=False, auto_delete=False | open-matchmaking.auth.token.refresh.direct  | Validates the refresh token, and provides back a new access token | A new JSON Web Token                       |
| auth.users.retrieve         | durable=True, passive=False, exclusive=False, auto_delete=False | open-matchmaking.auth.users.retrieve.direct | Returns an information about the current user                     | User                                       |
| auth.users.register         | durable=True, passive=False, exclusive=False, auto_delete=False | open-matchmaking.auth.users.register.direct | Create a new user for the game client                             | User or a validation error                 |

### Request and response examples
### Registering microservice
For registering a microservice a developer should send a message to the `auth.microservices.register` queue with content inside in JSON format and `content_type=application/json` in headers. The content must contain the following fields: 

| Field name | Parent | Description | Type |
|------------|--------|-------------|------|
| name | | Stores the name of the microservice. | String |
| version | | Specifies the version of the registered microservice. Must specified in the `major.minor.minor` format. | String |
| permissions | | Specifies a list of available permissions. | List of objects |
| codename | permissions[index].object | Defines the permissions name. Must specified in the `microservice.resource.action` format. | String |
| description | permissions[index].object | A short description of the certain permissions. | String |

An example of a request for registering:
```javascript
{
    'name': 'my-microservice',
    'version': '0.1.0',
    'permissions': [
        {
            'codename': 'my-microservice.stats.retrieve',
            'description': 'Can retrieve the player statistics'
        },
        {
            'codename': 'my-microservice.stats.update',
            'description': 'Can update the player statistics'
        }
    ]
}
```

### Creating a new JSON Web Token (JWT)
For creating a new JSON Web Token the user needs to send a message to the `auth.token.new` queue with a content that contains the following fields:

| Field name | Parent | Description                  | Type   |
|------------|--------|------------------------------|--------|
| username   |        | A user's login.              | String |
| password   |        | A password from the account. | String |

An example of the request from the reverse proxy:
```javascript
{
    'username': 'some_user',
    'password': 'super_secret_password'
}
```

An example of the response:
```javascript
{
    'access_token': 'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJhdXRoIiwiaWF0IjoxNTM4NTcwMTk0LCJleHAiOjE1NzAxMDYxOTUsImF1ZCI6IiIsInN1YiI6IiJ9.Z3zURZaWvX1p2B5dRqO1ma1-NC6Ip7blYIEyzylSi4s',
    'refresh_token': 'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJhdXRoIiwiaWF0IjoxNTM4NTcwMTk0LCJleHAiOjE1NzAxMDczMTEsImF1ZCI6IiIsInN1YiI6InJlZnJlc2gifQ.yg0zQtyxtHZ-nIxF5q70q-sph38sU5g5R6sMPt3qE6U'
}
```

### Verifying the JSON Web Token (JWT)
For verifying the generated JSON Web Token the user needs to send a message to the `auth.token.verify` queue with a content that contains the following fields:

| Field name   | Parent | Description                   | Type   |
|--------------|--------|-------------------------------|--------|
| access_token |        | A generated JSON Web Token.   | String |

An example of the request from the reverse proxy:
```javascript
{
    'access_token': 'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJhdXRoIiwiaWF0IjoxNTM4NTcwMTk0LCJleHAiOjE1NzAxMDYxOTUsImF1ZCI6IiIsInN1YiI6IiJ9.Z3zURZaWvX1p2B5dRqO1ma1-NC6Ip7blYIEyzylSi4s'
}
```

An example of the response:
```javascript
{
    'content': "OK",
    'valid': true
}
```

### Refreshing the JSON Web Token (JWT)
For refreshing the JSON Web Token the user needs to send a message to the `auth.token.refresh` queue with a content that contains the following fields:

| Field name    | Parent | Description                   | Type   |
|---------------|--------|-------------------------------|--------|
| access_token  |        | A generated JSON Web Token.   | String |
| refresh_token |        | A generated JSON Web Token which using for generating a new access JSON Web Token. | String |

An example of the request from the reverse proxy:
```javascript
{
    'access_token': 'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJhdXRoIiwiaWF0IjoxNTM4NTcwMTk0LCJleHAiOjE1NzAxMDYxOTUsImF1ZCI6IiIsInN1YiI6IiJ9.Z3zURZaWvX1p2B5dRqO1ma1-NC6Ip7blYIEyzylSi4s',
    'refresh_token': 'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJhdXRoIiwiaWF0IjoxNTM4NTcwMTk0LCJleHAiOjE1NzAxMDczMTEsImF1ZCI6IiIsInN1YiI6InJlZnJlc2gifQ.yg0zQtyxtHZ-nIxF5q70q-sph38sU5g5R6sMPt3qE6U'
}
```

An example of the response:
```javascript
{
    'access_token': 'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJhdXRoIiwiaWF0IjoxNTM4NTcyMDI2LCJleHAiOjE1NzAxMDgwMjYsImF1ZCI6IiIsInN1YiI6IiJ9.UmRDMP6ocNacTD7clmCWwtO74PvFRwgjnvvf6k-bYY0'
}
```

### Retrieving a user profile
For verifying the generated JSON Web Token the user needs to send a message to the `auth.users.retrieve` queue with a content that contains the following fields:

| Field name    | Parent | Description                   | Type   |
|---------------|--------|-------------------------------|--------|
| access_token  |        | A generated JSON Web Token.   | String |

An example of the request from the reverse proxy:
```javascript
{
    'access_token': 'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiIiLCJpYXQiOjE1Mzk2NzY2NDYsImV4cCI6MTU3MTIxMjY0NywiYXVkIjoiIiwic3ViIjoiIiwidXNlcl9pZCI6IjViYjRjOTg0MWY4ZWMwMDA0NTI0YTU0ZiJ9.3jUYf-ffQVJjdERnSYXApTAfvA3QLsiIQIWDQKHniZE'
}
```

An example of the response:
```javascript
{
    'id': '5bb4c9841f8ec0004524a54f',
    'username': 'some_user',
    'permissions': ['matchmaking.games.retrieve', 'matchmaking.games.update']
}
```

### Registering a new user
For verifying the generated JSON Web Token the user needs to send a message to the `auth.users.register` queue with a content that contains the following fields:

| Field name       | Parent | Description                 | Type   |
|------------------|--------|-----------------------------|--------|
| user             |        | A unique username.          | String |
| password         |        | A password for the account. | String |
| confirm_password |        | A password confirmation.    | String |

An example of the request from the reverse proxy:
```javascript
{
    'username': 'some_user',
    'password': '123456',
    'confirm_password': '123456'
}
```

An example of the response:
```javascript
{
    'id': '5bb4c9841f8ec0004524a54f',
    'username': 'some_user'
}
```
