# Matchmaking microservice

### Additional info
- [**Distributing tasks for a search**](../matchmaking.md#distributing-tasks-for-a-search)
- [**Saving the worker state**](../matchmaking.md#saving-the-worker-state)
- [**Searching games with considering player's location**](../matchmaking.md#searching-games-with-considering-players-location)
- [**Strategies of a searching players**](../matchmaking.md#strategies-of-a-searching-players)

### Why
- Necessary to have a separate microservice that processing requests related to searching players for a new match
- It should scaling up depending on the current load

### Benefits of using
- This microservice is independent and will be used only when it necessary to your project
- Configuring used strategies for this microservice brings the flexibility in describing the expected behaviour

### Dependencies
- [**Authorization / Authentication microservice**](auth-microservice.md)
- [**Player statistics microservice**](player-statistics-microservice.md)
- [**Strategist microservice**](strategist-microservice.md)
- [**Game servers pool**](game-servers-pool.md)

### API Endpoints
| Method | Endpoint | Usage | Returns |
|--------|----------|-------|---------|
|GET     | /matchmaking/api/health-check    | Health check endpoint for API Gateways                            | "OK" |

### RabbitMQ exchanges and queues 
#### Exchanges
| Exchange name                                     | Exchange type | Options                                        |
|---------------------------------------------------|---------------|------------------------------------------------| 
| open-matchmaking.matchmaking.generic-queue.direct | direct        | durable=True, passive=False, auto_delete=False |
| open-matchmaking.matchmaking.{name}.direct        | direct        | durable=True, passive=False, auto_delete=False |
| open-matchmaking.matchmaking.lobby.direct         | direct        | durable=True, passive=False, auto_delete=False |
| open-matchmaking.matchmaking.requeue.direct       | direct        | durable=True, passive=False, auto_delete=False |

#### Queues
| Queue name                      | Queue options                                                   | Exchange name                                     | Usage                                                                    | Returns                                                                                              |
|---------------------------------|-----------------------------------------------------------------|---------------------------------------------------|--------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------|
| matchmaking.games.search        | durable=True, passive=False, exclusive=False, auto_delete=False | open-matchmaking.direct                           | An entry point for searching a game with opponents                       | Validation error if was found. Otherwise passes the message to the "matchmaking.queue.generic" queue |
| matchmaking.queues.generic      | durable=True, passive=False, exclusive=False, auto_delete=False | open-matchmaking.matchmaking.generic-queue.direct | Extract the data about the player and send it to the next pipeline stage | -                                                                                                    |
| matchmaking.queues.{name}       | durable=True, passive=False, exclusive=False, auto_delete=False | open-matchmaking.matchmaking.{name}.direct        | Search opponents for a player with similar rating or a type              | -                                                                                                    |
| matchmaking.queues.lobbies      | durable=True, passive=False, exclusive=False, auto_delete=False | open-matchmaking.matchmaking.lobby.direct         | Prepares a game lobby for players and sends them invites into the game   | Connection details and credentials                                                                   |
| matchmaking.games.requeue       | durable=True, passive=False, exclusive=False, auto_delete=False | open-matchmaking.matchmaking.requeue.direct       | Requeue the player into the certain queue                                | -                                                                                                    |
| matchmaking.games.search.cancel | durable=True, passive=False, exclusive=False, auto_delete=False | open-matchmaking.direct                           | Cancel the searching a game for the player                               | "OK"                                                                                                 |

P.S. `{name}` is the "pattern" means that instead of this substitution should be specified a string, as the part of the full queue name  
P.S.S. All queues, except the `matchmaking.games.search` and `matchmaking.games.search.cancel` are **internal**               

### Messages
Because our microservice works with messages, we're expecting that each message will be with fixed amount of fields and will be defined in the concrete format, so that it will be possible to implement any desirable algorightm that will process the messages and will connect the players in one game lobby later.

All required headers must be specified for each message as they were defined in [protocol document](../protocol.md#headers).

#### Middleware queue
This queue is focused on the processing incoming messages, validating and passing them to the next processing stage if they are valid. Otherwise returns an error to the client.

For starting searching a new game a player must send a message to the `matchmaking.games.search` queue with content inside in JSON format and `content_type=application/json` in headers. The content contains the following fields: 

| Field name | Parent | Description | Type |
|------------|--------|-------------|------|
| game-mode | | The name of the game mode, where search will be executed. | String |

Example of the message content that will passed to the next stage:
```javascript
{
  "game-mode": "team-deathmatch",                   // Specified in the request body
  "id": "0146563d-0f45-4062-90a7-b13a583defad",     // Player ID (extracted from the message headers, `user_id`)
  "reply_to": "player-1-queue",                     // Response queue (exracted from the message headers, `reply_to`)
  "event-name": "find-game",                        // Event name (exracted from the message headers, `correlation_id`)
  "detail": {                                       // Contains the data extracted from the remote storage
    "rating": 2680,
    "content": {
      "id": "0146563d-0f45-4062-90a7-b13a583defad",
      "games": 531,
      "wins": 279,
      ...
    }
  }
}
```
This content of this message is relying on the fact that a data is extracted from the response to an external storage which is storing data about existing players in the system. The `game-mode` field is getting from the request by the player and passing into the message. The `rating` field is placed to the root of the message with the `content` field, that stores the detailed information about the player (which is also can store `rating` field as the duplicate). 

Example of a validation error:
```javascript
{
  "error": "The user does not have the necessary permissions for a resource."
}
```

#### Generic queue
This queue is storing messages about the active players in queue without any sorting by their skill or level. On this stage the processing node will send the send the message with the information about the player as-is to the certain queue, dependent on his/her rating or skill. 

Example of the message content that will passed to the next stage:
```javascript
{
  "game-mode": "team-deathmatch",
  "id": "0146563d-0f45-4062-90a7-b13a583defad",
  "reply_to": "player-1-queue",
  "event-name": "find-game",
  "detail": {                                       
    "rating": 2680,
    "content": {
      "id": "0146563d-0f45-4062-90a7-b13a583defad",
      "games": 531,
      "wins": 279,
      ...
    }
  }
}
```

#### Rating group queue
Handles the players and grouping them up into one game lobby, dependent on the used matchmaking algorithm. If processing node can't add the player to the certain game lobby, then requeueing the message for processing. Each an incoming message has the same format as for the generic queue.

Example of the data with the grouped players in one match:
```javascript
{
  "teams": {
    "team 1": [
      {
        "id": "0146563d-0f45-4062-90a7-b13a583defad",
        "response-queue": "player-1-queue",                   
        "event-name": "find-game",
        "detail": {
          "rating": 2680,
          "content": {
              "id": "0146563d-0f45-4062-90a7-b13a583defad",
              "games": 531,
              "wins": 279,
              ...
          }
        }
      },
      { 
        "id": "a5d1d7a1-7b80-41fe-8bbc-441dba86a18a",
        "response-queue": "player-2-queue",
        "event-name": "find-game",
        "detail": {
          "rating": 2721,
          "content": {
              "id": "a5d1d7a1-7b80-41fe-8bbc-441dba86a18a",
              "games": 150,
              "wins": 72,
              ...
          }
        }
      }
      ...
    ],
    "team 2": [
      { 
        "id": "56acbeb4-687d-4c8b-a881-0b9abdda64e4",
        "response-queue": "player-7-queue",
        "event-name": "find-game",
        "detail": {
          "rating": 2705,
          "content": {
              "id": "56acbeb4-687d-4c8b-a881-0b9abdda64e4",
              "games": 61,
              "wins": 35,
              ...
          }
        }
      },
      { 
        "id": "1b23b523-dd19-4a8c-8749-9b20588c962a",
        "response-queue": "player-8-queue",
        "event-name": "find-game",
        "detail": {
          "rating": 2783,
          "content": {
              "id": "1b23b523-dd19-4a8c-8749-9b20588c962a",
              "games": 221,
              "wins": 127,
              ...
          }
        }
      }
      ...
    ],
    ...
  }
  "game-mode": "team-deathmatch"
}
```
From the some point of view it is a denormalized data, which is have a sense for us, because all of those data can be used for handling special cases, balancing players for the match and so on. The data time-to-time will be written to the external storage for make it durable and sustainable to sudden fails.

Each player, which is can be grouped up the players in one lobby and will be transferred to next processing stage is defined in the following format:
```javascript
{
  "id": "0146563d-0f45-4062-90a7-b13a583defad",
  "reply_to": "player-1-queue",
  "event-name": "find-game"
}
```
As you can see here, we have dropped almost everything that were used for searching players. But on the next stage of the processing we don't need the additional information about each player, because has a knowledge about the formed groups.

#### Queue for search retries / "Requeue" queue
Stores the players as they were in certain matchmaking queues and put them into generic queue. Each message has the same format as for the generic/certain matchmaking queues.

#### Game lobby queue 
Stores the prepared list of the players (which are splitted onto teams), that must be connected in one game to the game server. Creates a new game server (or getting one from the existing servers), a send the message to each client to connect to the certain server. 

Example of the incoming message:
```javascript
{
  "teams": {
    "team 1": [
      {
        "id": "0146563d-0f45-4062-90a7-b13a583defad",
        "response-queue": "player-1-queue",
        "event-name": "find-game"
      },
      { 
        "id": "a5d1d7a1-7b80-41fe-8bbc-441dba86a18a",
        "response-queue": "player-2-queue",
        "event-name": "find-game"
      }
      ...
    ],
    "team 2": [
      { 
        "id": "56acbeb4-687d-4c8b-a881-0b9abdda64e4",
        "response-queue": "player-7-queue",
        "event-name": "find-game"
      },
      { 
        "id": "1b23b523-dd19-4a8c-8749-9b20588c962a",
        "response-queue": "player-8-queue",
        "event-name": "find-game"
      }
      ...
    ],
    ...
  }
  "game-mode": "team-deathmatch"
}
```

Example of the message to each client:
```javascript
{
  "content": {
    "ip": "127.0.0.1",
    "port": "9001",
    "team": "team 1",
    "credentials": {
      ...             // Login with password, tokens, etc.
    }
  },
  "event-name": "find-game"
}
```

#### Queue for requests to cancel searching a new game
This queue is used for cancelling the search process a new game for the player. Regardless of the actual state of the player in the search queue returns the `"OK"` in the response body.  
All what is necessary to cancel the search process is to specify the `user_id` header with the UUID4 of the player. Any data, specified in the body of the request will be ignored and won't be used.
