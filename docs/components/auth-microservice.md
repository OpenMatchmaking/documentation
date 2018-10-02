# Authorization / Authentication microservice

### Why 
- Get an opportunity to restrict users, so that they can't send requests to microservices without required credentials and permissions.
- Provide for developers the solution for cases when OpenMatchmaking should work with the dedicated authorization/authentication server, withot communicating with existing authorization/authentication servers.
- Necessary to store an information about users and let them to get (or refresh) an access token.
- Required a storage for microservices permissions, so that they can be assigned to each existing user

### Benefits of using
- Can be used as the 3rd party authorization/authentication service for restricting an access in a pair with [reverse proxy](https://github.com/OpenMatchmaking/documentation/blob/master/docs/components/reverse-proxy.md#reverse-proxy)
- A good choice for cases when OpenMatchmaking should work independently, without communicating with other services (not related to OpenMatchmaking)

### General information
General principles of auth/auth microservice can be shown with the following picture:

<p align="center">
  <img src="https://github.com/OpenMatchmaking/documentation/blob/master/docs/images/microservice-auth-schema.png"/>
</p>

As you can see on the picture, the microservice is communicating with three different entities with using storage as a database for a some data:
- Game client is authenticating for getting an access and a refresh tokens  
- Reverse proxy is going to use this microservice for verifying the given token from the game client and receiving a user data that will be used per each request before sending it to the certain message queue
- Internal parts of Open Matchmaking which are communicating via specially created message queues for auth/auth microservice. It's a hidden part, a **don't provide** any public APIs for it. Only for internal usage

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
|codename    | VARCHAR    | NOT NULL         | A permission name in format "{microservice}.{resource}.{action}" |
|description | VARCHAR    |                  | Human-readable description about the permission                  |

#### Microservice table
| Field name | Field type | Constraints      | Description                                              |
|------------|------------|------------------|----------------------------------------------------------|
|id          | UUID       | PK               | A database unique object identifier                      |
|name        | VARCHAR    | UNIQUE, NOT NULL | A unique Microservice identifier                         |
|version     | VARCHAR    | NOT NULL         | Microservice version in format "{major}.{minor}.{patch}" |
|permissions | Relation   |                  | One-To-Many Relationship to Permission                   |

### API Endpoints
| Method | Endpoint | Usage | Returns |
|--------|----------|-------|---------|
|GET     | /auth/api/health-check    | Health check endpoint for API Gateways                            | "OK" |
|POST    | /auth/api/token/new       | Get an access token (or an access token and refresh token)        | A new JSON Web Token or a validation error |
|POST    | /auth/api/token/verify    | Returns with whether or not a given access token is valid         | {"is_valid": true} or a validation error | result |
|POST    | /auth/api/token/refresh   | Validates the refresh token, and provides back a new access token | A new JSON Web Token |
|GET     | /auth/api/v1/users/me     | Returns an information about the current user                     | User |
|POST    | /auth/api/v1/users/game-client/register | Create a new user for the game client               | User or a validation error |

### RabbitMQ exchanges and queues 
#### Queues
| Queue name                  | Queue options                                                   | Exchange name           | Usage                                        | Returns                    |
|-----------------------------|-----------------------------------------------------------------|-------------------------|----------------------------------------------|----------------------------|
| auth.microservices.register | durable=True, passive=False, exclusive=False, auto_delete=False | open-matchmaking.direct | Register a new microservice with permissions | "OK" or a validation error |


### Registering microservice
For registering a microservice a developer should send a message to the `auth.microservices.register` queue with content inside in JSON format and `content_type=application/json` in headers. The content must contain the following fields: 

| Field name | Parent | Description | Type |
|------------|--------|-------------|------|
| name | | Stores the name of the microservice. | String |
| version | | Specifies the version of the registered microservice. Must specified in the `major.minor.minor` format. | String |
| permissions | | Specifies a list of available permissions. | List of objects |
| codename | permissions[index].object | Defines the permissions name. Must specified in the `microservice.resource.action` format. | String |
| description | permissions[index].object | A short description of the certain permissions. | String |

#### Example of a request for registering
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
