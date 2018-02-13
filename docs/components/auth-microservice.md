# Authorization / Authentication microservice

### Why 
- Get an opportunity to restrict users, so that they can't send requests to microservices without required credentials and permissions.
- Provide for developers the solution for cases when OpenMatchmaking should work with the dedicated authorization/authentication server, withot communicating with existing authorization/authentication servers.
- Necessary to store an information about users and let them to get (or refresh) an access token.

### Benefits of using
- Can be used as the 3rd party authorization/authentication service for restricting an access in a pair with [reverse proxy](https://github.com/OpenMatchmaking/documentation/blob/master/docs/components/reverse-proxy.md#reverse-proxy)
- A good choice for cases when OpenMatchmaking should work independently, without communicating with other services (not related to OpenMatchmaking)

### Entity relationship diagram

### API Endpoints
| Method | Endpoint | Usage | Returns |
|--------|----------|-------|---------|
|GET     | /api/healthcheck         | Health check endpoint for API Gateways                     | - |
|POST    | /api/token               | Get an access token (or an access token and refresh token) | A new JSON Web Token |
|GET     | /api/v1/users            | Get list of users                                          | Users |
|POST    | /api/v1/users            | Create a new user                                          | User |
|GET     | /api/v1/users/{id}       | Get detail information about the user                      | User |
|PUT     | /api/v1/users/{id}       | Update an information about the user                       | User |
|DELETE  | /api/v1/users/{id}       | Remove the user                                            | - |
|GET     | /api/v1/groups           | Get list of available groups                               | Groups |
|POST    | /api/v1/groups           | Create a new group                                         | Group |
|GET     | /api/v1/groups/{id}      | Get a detail information about the group                   | Group |
|PUT     | /api/v1/groups/{id}      | Create a new group                                         | Group |
|DELETE  | /api/v1/groups/{id}      | Remove the group                                           | - |
|GET     | /api/v1/permissions      | Get list of permissions                                    | Permissions |
|POST    | /api/v1/permissions      | Create a new permission                                    | Permission |
|GET     | /api/v1/permissions/{id} | Get a detail information about the permission              | Permission |
|PUT     | /api/v1/permissions/{id} | Update an information about the permission                 | Permission |
|DELETE  | /api/v1/permissions/{id} | Remove the permission                                      | - |

Endpoint:  
URL:  
Description:  
Request parameters:  
Example of response:  
