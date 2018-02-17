# Authorization / Authentication microservice

### Why 
- Get an opportunity to restrict users, so that they can't send requests to microservices without required credentials and permissions.
- Provide for developers the solution for cases when OpenMatchmaking should work with the dedicated authorization/authentication server, withot communicating with existing authorization/authentication servers.
- Necessary to store an information about users and let them to get (or refresh) an access token.
- Required a storage for microservices permissions, so that they can be assigned to each existing user

### Benefits of using
- Can be used as the 3rd party authorization/authentication service for restricting an access in a pair with [reverse proxy](https://github.com/OpenMatchmaking/documentation/blob/master/docs/components/reverse-proxy.md#reverse-proxy)
- A good choice for cases when OpenMatchmaking should work independently, without communicating with other services (not related to OpenMatchmaking)

### Entity relationship diagram
<p align="center">
  <img src="https://github.com/OpenMatchmaking/documentation/blob/master/docs/images/microservice-auth-db.png"/>
</p>

**IMPORTANT**: The tables "Group", "Permission" and "Microservice" will be updated by other part of microservice, that will be handling requests from inner microservices after their instantiating.

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
|codename    | VARCHAR    | NOT NULL         | A permission name in format "{microservice}.{resource}.{action}" |
|description | VARCHAR    |                  | Human-readable description about the permission                  |

#### Microservice table
| Field name | Field type | Constraints      | Description                                  |
|------------|------------|------------------|----------------------------------------------|
|id          | UUID       | PK               | A database unique object identifier          |
|name        | VARCHAR    | UNIQUE, NOT NULL | A unique Microservice identifier             |
|version     | VARCHAR    | NOT NULL         | Microservice version in format "{x}.{y}.{z}" |
|permissions | Relation   |                  | One-To-Many Relationship to Permission       |

### API Endpoints
| Method | Endpoint | Usage | Returns |
|--------|----------|-------|---------|
|GET     | /auth/api/health-check    | Health check endpoint for API Gateways                            | - |
|POST    | /auth/api/token           | Get an access token (or an access token and refresh token)        | A new JSON Web Token |
|POST    | /auth/api/verify          | Returns with whether or not a given access token is valid         | A token validation | result |
|POST    | /auth/api/refresh         | Validates the refresh token, and provides back a new access token | A new JSON Web Token |
|POST    | /auth/api/v1/users/me     | Returns an information about the current user                     | User |
|POST    | /auth/api/v1/users/game-client/register | Create a new user for the game client               | User |

### RabbitMQ queues
| Queue name                | Usage                                        | Returns                  |
|---------------------------|----------------------------------------------|--------------------------|
|auth.microservices.register| Register a new microservice with permissions | "OK" or validation error |
