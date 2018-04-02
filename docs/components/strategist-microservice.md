# Strategist microservice

This microservice is specialized on forming up the groups of players and building fair matches for each player.

### Additional info
- [**Strategies of a searching players**](https://github.com/OpenMatchmaking/documentation/blob/master/docs/matchmaking.md#strategies-of-a-searching-players)

### Why 
- Elixir/Erlang libraries for integrating with Python interpreters not enough flexible for unusual use cases
- Impossible to pass a large JSON (or in any other suitable data format) for further processing. Necessary to create temporary files,shared memory, etc. 
- Bidirectional serializing/deserializing data can be an issue during the processing large amount of data.
- Large support overheads for microservices that works together in one container, but different environments.

### Benefits of using
- Used code is supported only by a small team of developers
- Can be scaled independently from other microservice
- A minimal overhead on intergrating different microservices

### RabbitMQ queues
| Queue name             | Exchange name                       | Usage                                             | Returns                         |
|------------------------|-------------------------------------|---------------------------------------------------|---------------------------------|
| strategist.match.сheck | open-matchmaking. strategist.check.fanout | Checks if the player can participate in the match | Updated list of grouped players |


### Messages
Because our microservice works with messages, we're expecting that each message will be with fixed amount of fields and will be defined in the concrete format, so that it will be possible to implement any desirable algorightm that will process the messages and will connect the players in one game lobby later.

All required headers must be specified for each message as they were defined in [protocol document](https://github.com/OpenMatchmaking/documentation/blob/master/docs/protocol.md#headers).

#### Check queue
This queue is storing messages where each message contains an information about the used game mode for a new game, list of grouped players for the match and a new player which can be potentially added to this list of players.

Example of the message content that can be received by this microservice:
```javascript
{
  "game-mode": "1v1",
  "new-player": {
    "id": "0146563d-0f45-4062-90a7-b13a583defad",
    "response-queue": "player-2-queue",
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
  "grouped-players": {    
    "team 1": [
      {
        "id": "56acbeb4-687d-4c8b-a881-0b9abdda64e4",
        "response-queue": "player-1-queue",
        "event-name": "find-game",
        "detail": {
          "rating": 2702,
          "content": {
            "id": "56acbeb4-687d-4c8b-a881-0b9abdda64e4",
            "games": 161,
            "wins": 91,
            ...
          }
        }
      }
    ],
    "team 2": []
  }
}
```
The data for the `game-mode`, `new-player` and `grouped-players` fields are passed as is by the matchmaking microservice on the one of the processing stages.

Example of the response:
```javascript
{
  "added": true,
  "grouped-players": {    
    "team 1": [
      {
        "id": "56acbeb4-687d-4c8b-a881-0b9abdda64e4",
        "response-queue": "player-1-queue",
        "event-name": "find-game",
        "detail": {
          "rating": 2702,
          "content": {
            "id": "56acbeb4-687d-4c8b-a881-0b9abdda64e4",
            "games": 161,
            "wins": 91,
            ...
          }
        }
      }
    ],
    "team 2": [
      "id": "0146563d-0f45-4062-90a7-b13a583defad",
      "response-queue": "player-2-queue",
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
    ]
  }
}
```
