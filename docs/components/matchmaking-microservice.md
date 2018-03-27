# Matchmaking microservice

### Additional info
- [**Distributing tasks for a search**](https://github.com/OpenMatchmaking/documentation/blob/master/docs/matchmaking.md#distributing-tasks-for-a-search)
- [**Strategies of a searching players**](https://github.com/OpenMatchmaking/documentation/blob/master/docs/matchmaking.md#strategies-of-a-searching-players)

### Why
- Necessary to have a separate microservice that processing requests related to searching players for a new match
- It should scaling up depending on the current load

### Benefits of using
- This microservice is independent and will be used only when it necessary to your project
- Configuring used strategies for this microservice brings the flexibility in describing the expected behaviour

### RabbitMQ queues
| Queue name                | Exchange name           | Usage                                      | Returns                          |
|---------------------------|-------------------------|--------------------------------------------|----------------------------------|
| matchmaking.games.search  | open-matchmaking.direct | An entry point for searching a game with opponents | Validation error if was found. Otherwise passes the message to the "matchmaking.queue.generic" queue | No |
| matchmaking.queues.generic        | open-matchmaking.  matchmaking.generic-queue.fanout | Extract the data about the player and send it to the next pipeline stage | - |
| matchmaking.queues.{name}  | open-matchmaking.  matchmaking.{name}.fanout | Search opponents for a player with similar rating or a type               | - |              
| matchmaking.queues.lobbies        | open-matchmaking.  matchmaking.lobby.fanout | Prepares a game lobby for players and sends them invites into the game    | Connection details and credentials |
| matchmaking.games.requeue         | open-matchmaking.  matchmaking.requeue.direct | Requeue the player into the certain queue                                 | - |

P.S. `{name}` is the "pattern" means that instead of this substitution should be specified a string, as the part of the full queue name  
P.S.S. All queues, except the "matchmaking.games.search" are **internal**                       

### Messages
Because our microservice works with messages, we're expecting that each message will be with fixed amount of fields and will be defined in the concrete format, so that it will be possible to implement any desirable algorightm that will process the messages and will connect the players in one game lobby later.

All required headers must be specified for each message as they were defined in [protocol document](https://github.com/OpenMatchmaking/documentation/blob/master/docs/protocol.md#headers).

#### Middleware queue
This queue is focused on the processing incoming messages, validating and passing them to the next processing stage if they are valid. Otherwise returns an error to the client.

Example of the message content that will passed to the next stage:
```javascript
{
  "game-mode": "team-deathmatch",
  "rating": 2680,
  "content": {
    "id": "0146563d-0f45-4062-90a7-b13a583defad",
    "games": 531,
    "wins": 279,
    ...
  }
}
```
This content of this message is relying on the fact that a data is extracted from the response to an external storage which is storing data about existing players in the system. The `game-mode` fields is getting from the request by the player and passing into the message. The `rating` field is placed to the root of the message with the `content` field, that stores the detailed information about the player (which is also can store `rating` field as the duplicate). 

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
  "rating": 2680,
  "content": {
    "id": "0146563d-0f45-4062-90a7-b13a583defad",
    "games": 531,
    "wins": 279,
    ...
  }
}
```

#### {Rating group} queue
Handles the players and grouping them up into one game lobby, dependent on the used matchmaking algorithm. If processing node can't add the player to the certain game lobby, then requeueing the message for processing. Each message has the same format as for the generic queue.

#### Queue for search retries / "Requeue" queue
Stores the players as they were in certain matchmaking queues and put them into generic queue. Each message has the same format as for the generic/certain matchmaking queues.

Each player, which is can be grouped up the players in one lobby, defined in the following format:
```javascript
{
    "0146563d-0f45-4062-90a7-b13a583defad",  // Player ID
    "reply_to": "player-1-queue",            // Response queue (exracted from the message headers)
    "event-name": "find-game"                // Event name (exracted from the message headers)
}
```

#### Game lobby queue 
Stores the prepared list of the players (which are splitted onto teams), that must be connected in one game to the game server. Creates a new game server (or getting one from the existing servers), a send the message to each client to connect to the certain server. 

Example of incoming message:
```javascript
{
    "team 1": [
        {
            "id": "0146563d-0f45-4062-90a7-b13a583defad",
            "reply_to": "player-1-queue",
            "event-name": "find-game"
        },
        { 
            "id": "a5d1d7a1-7b80-41fe-8bbc-441dba86a18a",
            "reply_to": "player-2-queue",
            "event-name": "find-game"
        }
        ...
    ],
    "team 2": [
        { 
            "id": "56acbeb4-687d-4c8b-a881-0b9abdda64e4",
            "reply_to": "player-7-queue",
            "event-name": "find-game"
        },
        { 
            "id": "1b23b523-dd19-4a8c-8749-9b20588c962a",
            "reply_to": "player-8-queue",
            "event-name": "find-game"
        }
        ...   
    ]
}
```

Example of the message to each client:
```javascript
{
    "ip": "127.0.0.1",
    "port": "9001"
    "credentials": {
        ...             // Login with password, tokens and etc.
    }
}    
```
