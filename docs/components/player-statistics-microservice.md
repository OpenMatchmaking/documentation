# Player statistics microservice

This microservice is specialized on storing statistics data per for each existing player in Open Matchmaking platform.

### Why 
- Requires a storage for the data per each player, separately from Authorization / Authentication microservice
- Necessary to track a player progress after each game and perform certain actions on this data 

### Benefits of using
- The data can be easily denormalized depends on your use cases
- The stored data can be used for analytics and tracking for a state of the game (balance, the most picked characters and so on)

### Dependencies
- [**Authorization / Authentication microservice**](https://github.com/OpenMatchmaking/documentation/blob/master/docs/components/auth-microservice.md)

### Entity relationship diagram
#### Statistic table
| Field name      | Field type | Constraints | Description                                |
|-----------------|------------|-------------|--------------------------------------------|
| player_id       | UUID       | PK          | A database unique object identifier (same as in Authorization / Authentication microservice) |
| games           | Integer    | NOT NULL    | Total number of games played                       |
| wins            | Integer    | NOT NULL    | Total number of games when the player won          |
| loses           | Integer    | NOT NULL    | Total number of games when the player was defeated |
| rating          | Integer    | NOT NULL    | An actual rating of the player                     |

Also the table can have an extra fields in database (for example, extra columns/documents in NoSQL or a nested JSON/HStore field in RDBMS) that can be used for providing for a detail information about the player.

### API Endpoints
| Method | Endpoint | Usage | Returns |
|--------|----------|-------|---------|
|GET     | /auth/api/health-check    | Health check endpoint for API Gateways                            | "OK" |

### RabbitMQ exchanges and queues 
#### Exchanges
| Exchange name                                           | Exchange type | Options                                        |
|---------------------------------------------------------|---------------|------------------------------------------------| 
| open-matchmaking.player-stats.statistic.init.direct     | direct        | durable=True, passive=False, auto_delete=False |
| open-matchmaking.player-stats.statistic.retrieve.direct | direct        | durable=True, passive=False, auto_delete=False |
| open-matchmaking.player-stats.statistic.update.direct   | direct        | durable=True, passive=False, auto_delete=False |

#### Queues
| Queue name                      | Queue options                                                   | Exchange name                                           | Usage                                                         | Returns                          |
|---------------------------------|-----------------------------------------------------------------|---------------------------------------------------------|---------------------------------------------------------------|----------------------------------|
| player-stats.statistic.init     | durable=True, passive=False, exclusive=False, auto_delete=False | open-matchmaking.player-stats.statistic.init.direct     | Initializes statistics from an empty state for the new player | Statistics or a validation error |
| player-stats.statistic.retrieve | durable=True, passive=False, exclusive=False, auto_delete=False | open-matchmaking.player-stats.statistic.retrieve.direct | Returns the player statistics                                 | Statistics or a validation error |
| player-stats.statistic.update   | durable=True, passive=False, exclusive=False, auto_delete=False | open-matchmaking.player-stats.statistic.update.direct   | Updates the player statistics                                 | Statistics or a validation error |

### Request and response examples
#### Initializing player statistics
For initializing and resetting the player statistics a developer should send a message to the `player-stats.statistic.init` queue with content inside in JSON format and `content_type=application/json` in headers. The message content contains the following fields:

| Field name | Parent | Description | Type |
|------------|--------|-------------|------|
| player_id | | Stores a unique identifier of the player. | UUID as string |

An example of the request between microservices directly
```json
{
  "player_id": "507f1f77aabbccd000000001"
}
```

An example of the response:
```json
{
  "content": {
    "id": "507f1f77aabbccd000000000", 
    "player_id": "507f1f77aabbccd000000001", 
    "total_games": 0, 
    "wins": 0, 
    "loses": 0, 
    "rating": 0
  },
  "event-name": "init-player"
}
```

#### Retrieving player statistics
For getting the player statistics a developer should send a message to the `player-stats.statistic.retrieve` queue with content inside in JSON format and `content_type=application/json` in headers. The message content contains the following fields:

| Field name | Parent | Description | Type |
|------------|--------|-------------|------|
| player_id | | Stores a unique identifier of the player. | UUID as string |

An example of the request between microservices directly:
```json
{
  "player_id": "507f1f77aabbccd000000001"
}
```

An example of the response:
```json
{
  "content": {
    "id": "507f1f77aabbccd000000000", 
    "player_id": "507f1f77aabbccd000000001", 
    "total_games": 10, 
    "wins": 6, 
    "loses": 4, 
    "rating": 2676
  },
  "event-name": "init-player"
}
```

#### Updating player statistics
For updating the player statistics a developer should send a message to the `player-stats.statistic.update` queue with content inside in JSON format and `content_type=application/json` in headers. The message content contains the following fields:

| Field name | Parent | Description | Type |
|------------|--------|-------------|------|
| player_id | | Stores a unique identifier of the player. | UUID as string |
| total_games | | Total amount of games, already finished by the player. | Integer |
| wins | | Total number of wins. | Integer |
| loses | | Total number of defeats. | Integer |
| rating | | Current rating in competitive mode. | Integer |

An example of the request between microservices directly:
```json
{
  "player_id": "507f1f77aabbccd000000001",
  "total_games": 11,
  "wins": 6,
  "loses": 5,
  "rating": 2648
}
```

An example of the response:
```json
{
  "content": {
    "id": "507f1f77aabbccd000000000", 
    "player_id": "507f1f77aabbccd000000001", 
    "total_games": 11, 
    "wins": 6, 
    "loses": 5, 
    "rating": 2648
  },
  "event-name": "init-player"
}
```

