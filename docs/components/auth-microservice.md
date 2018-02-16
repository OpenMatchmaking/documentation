# Authorization / Authentication microservice

### Why 
- Get an opportunity to restrict users, so that they can't send requests to microservices without required credentials and permissions.
- Provide for developers the solution for cases when OpenMatchmaking should work with the dedicated authorization/authentication server, withot communicating with existing authorization/authentication servers.
- Necessary to store an information about users and let them to get (or refresh) an access token.

### Benefits of using
- Can be used as the 3rd party authorization/authentication service for restricting an access in a pair with [reverse proxy](https://github.com/OpenMatchmaking/documentation/blob/master/docs/components/reverse-proxy.md#reverse-proxy)
- A good choice for cases when OpenMatchmaking should work independently, without communicating with other services (not related to OpenMatchmaking)

### Entity relationship diagram
<p align="center">
  <img src="https://github.com/OpenMatchmaking/documentation/blob/master/docs/images/microservice-auth-db.png"/>
</p>

### API Endpoints
| Method | Endpoint | Usage | Returns |
|--------|----------|-------|---------|
|GET     | /api/health-check    | Health check endpoint for API Gateways                            | - |
|POST    | /api/auth/token      | Get an access token (or an access token and refresh token)        | A new JSON Web Token |
|POST    | /api/auth/verify     | Returns with whether or not a given access token is valid         | A token validation result |
|POST    | /api/auth/refresh    | Validates the refresh token, and provides back a new access token | A new JSON Web Token |
|POST    | /api/auth/me         | Returns an information about the current user                     | User |
|POST    | /api/v1/users/game-client/register | Create a new user for the game client               | User |
|POST    | /api/v1/users/game-server/register | Create a new user for the game server               | User |
