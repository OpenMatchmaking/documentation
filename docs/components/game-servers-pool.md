# Game servers pool

### Why 
- Necessary to track which game servers are available for new games
- Different game servers can have different ways to authenticate (tokens, passwords, etc.)
- Game servers can be served in different clouds (Amazon Web Services, Google Cloud, Microsoft Azure, etc.)

### Benefits of using
- All inner logic around instantiating new game servers for endusers is hidden
- Game server only registers or updates the existing information about itself
- Can be independently scaled up 

### Dependencies
- [**Authorization / Authentication microservice**](https://github.com/OpenMatchmaking/documentation/blob/master/docs/components/auth-microservice.md)

### Entity relationship diagram
#### Server table
| Field name      | Field type | Constraints | Description                                |
|-----------------|------------|-------------|--------------------------------------------|
| id              | UUID       | PK          | A database unique object identifier        |
| host            | VARCHAR    | NOT NULL    | IP address of the server                   |
| port            | Integer    | NOT NULL    | The listened port by the server            |
| available_slots | Integer    | NOT NULL    | Number of available slots for players      |
| credentials     | JSON       | NOT NULL    | The data used for connecting to the server |
| game_mode       | VARCHAR    | NOT NULL    | Type of served games                       |

### RabbitMQ exchanges and queues 
#### Queues
| Queue name                        | Queue options                                                   | Exchange name           | Usage                                    | Returns                                  |
|-----------------------------------|-----------------------------------------------------------------|-------------------------|------------------------------------------|------------------------------------------|
| game-servers-pool.server.register | durable=True, passive=False, exclusive=False, auto_delete=False | open-matchmaking.direct | Register a new game server               | A unique server ID or a validation error |
| game-servers-pool.server.retrieve | durable=True, passive=False, exclusive=False, auto_delete=False | open-matchmaking.direct | Get a server with credentials to connect | Server with credentials                  |

### Request and response examples

#### Registering a new server
Request:
```json
{
  "host": "127.0.0.1",
  "port": "5000",
  "available_slots": 100,
  "credentials": {
      "token": "secret_token"
  },
  "game_mode": "team-deathmatch",
}
```

Response (success):
```json
{
  "content": {
      "id": "b188eb61-d19b-4c6b-b73b-ee744fda61c6"
  },
  "event-name": "register-new-server"
}
```

Response (error):
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
    "required_slots": 10,
    "game_mode": "team-deathmatch"
  },
  "token": "a unique token",
  "event-name": "get-server"
}
```
- Between microservices
```json
{
  "required_slots": 10,
  "game_mode": "team-deathmatch"
}
```

Response (success):
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

Response (error):
```json
{
  "error": {
      "type": "ValidationError",
      "details": {
          "required_slots": ["Field can't be null."]
      }
  },
  "event-name": "get-server"
}
```
