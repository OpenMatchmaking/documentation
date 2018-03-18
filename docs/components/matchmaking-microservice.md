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
| matchmaking.games.search  | open-matchmaking  .direct | An entry point for searching a game with opponents | Validation error if was found. Otherwise passes the message to the "matchmaking.queue.generic" queue | No |
| matchmaking.queues.generic        | open-matchmaking.  matchmaking.generic-queue.fanout | Extract the data about the player and send it to the next pipeline stage | - |
| matchmaking.queues.{name}  | open-matchmaking.  matchmaking.{name}.fanout | Search opponents for a player with similar rating or a type               | - |              
| matchmaking.queues.lobbies        | open-matchmaking.  matchmaking.lobby.fanout | Prepares a game lobby for players and sends them invites into the game    | Connection details and credentials |
| matchmaking.games.requeue         | open-matchmaking.  matchmaking.direct | Requeue the player into the certain queue                                 | - |

P.S. `{name}` is the "pattern" means that instead of this substitution should be specified a string, as the part of the full queue name  
P.S.S. All queues, except the "matchmaking.games.search" are **internal**                       
