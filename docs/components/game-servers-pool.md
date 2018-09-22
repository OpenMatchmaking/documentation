# Game servers pool

### Why 
- Necessary to track which game servers are available for new games
- Different game servers can have different ways to authenticate (tokens, passwords, etc.)
- Game servers can be served in different clouds (Amazon Web Services, Google Cloud, Microsoft Azure, etc.)

### Benefits of using
- All inner logic around instantiating new game servers for endusers is hidden
- Game server only registers or updates the existing information about itself
- Can be scaled up independently 

### Dependencies
- [**Authorization / Authentication microservice**](https://github.com/OpenMatchmaking/documentation/blob/master/docs/components/auth-microservice.md)

### Entity relationship diagram
#### Server table
| Field name      | Field type  | Constraints | Description                                |
|-----------------|-------------|-------------|--------------------------------------------|
| id              | UUID        | PK          | A database unique object identifier        |
| host            | VARCHAR     | NOT NULL    | IP address of the server                   |
| port            | Integer     | NOT NULL    | The listened port by the server            |
| available_slots | Integer     | NOT NULL    | Number of available slots for players      |
| credentials     | JSON (dict) | NOT NULL    | The data used for connecting to the server |
| game_mode       | VARCHAR     | NOT NULL    | Type of served games                       |

### API Endpoints
| Method | Endpoint | Usage | Returns |
|--------|----------|-------|---------|
|GET     | /auth/api/health-check    | Health check endpoint for API Gateways                            | "OK" |

### RabbitMQ exchanges and queues 
#### Queues
| Queue name                        | Queue options                                                   | Exchange name                                            | Usage                                                           | Returns                                  |
|-----------------------------------|-----------------------------------------------------------------|----------------------------------------------------------|-----------------------------------------------------------------|------------------------------------------|
| game-servers-pool.server.register | durable=True, passive=False, exclusive=False, auto_delete=False | open-matchmaking.game-server-pool.server.register.direct | Registers a new game server or updates all information about it | A unique server ID or a validation error |
| game-servers-pool.server.retrieve | durable=True, passive=False, exclusive=False, auto_delete=False | open-matchmaking.game-server-pool.server.retrieve.direct | Gets a server with credentials to connect                       | Server with credentials                  |
| game-servers-pool.server.update   | durable=True, passive=False, exclusive=False, auto_delete=False | open-matchmaking.game-server-pool.server.update.direct   | Updates an infomation about available slots for games           | The updated information about the server |

### Request and response examples

#### Registering a new server / Updating the information about the server
A request for registering a new server:
```json
{
  "host": "127.0.0.1",
  "port": "5000",
  "available-slots": 100,
  "credentials": {
      "token": "secret_token"
  },
  "game-mode": "team-deathmatch"
}
```

A request for updating an information about the server (when the server knows his the unique ID after registering):
```
{
  "id": "b188eb61-d19b-4c6b-b73b-ee744fda61c6",
  "host": "127.0.0.1",
  "port": "5000",
  "available-slots": 150,
  "credentials": {
      "token": "secret_token"
  },
  "game-mode": "team-deathmatch"
}
```

An example of the response for the valid request:
```json
{
  "content": {
      "id": "b188eb61-d19b-4c6b-b73b-ee744fda61c6"
  },
  "event-name": "register-new-server"
}
```

An example of the response with the validation error:
```json
{
  "error": {
      "type": "ValidationError",
      "details": {
          "host": ["Field can't be empty."]
      }
  },
  "event-name": "register-new-server"
}
```

#### Receiving a server with credentials 
Request:
- By a client
```json
{
  "url": "api/v1/game-servers-pool/retrieve",
  "content": {
    "required-slots": 10,
    "game-mode": "team-deathmatch"
  },
  "token": "a unique token",
  "event-name": "get-server"
}
```
- Between microservices
```json
{
  "required-slots": 10,
  "game-mode": "team-deathmatch"
}
```

An example of the response (success):
```json
{
  "content": {
      "host": "127.0.0.1",
      "port": 5000,
      "credentials": {
          "token": "secret_token"
      }
  },
  "event-name": "get-server"
}
```

An example of the response with the validation error:
```json
{
  "error": {
      "type": "ValidationError",
      "details": {
          "required-slots": ["Field can't be null."]
      }
  },
  "event-name": "get-server"
}
```

#### Updating an information about available slots for games (only between internal microservices)
```json
{
  "id": "b188eb61-d19b-4c6b-b73b-ee744fda61c6",
  "freed-slots": 10
}
```

An example of the response for the valid request:
```json
{
  "content": {
      "id": "b188eb61-d19b-4c6b-b73b-ee744fda61c6",
      "available-slots": 110
  },
  "event-name": "register-new-server"
}
```

An example of response with the validation error:
```json
{
  "error": {
      "type": "ValidationError",
      "details": {
          "freed-slots": ["Field can't be empty."]
      }
  },
  "event-name": "register-new-server"
}
```
