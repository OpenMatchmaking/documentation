# Player statistics microservice

This microservice is specialized on storing statistics data per for each existing player in Open Matchmaking platform.

### Why 
- Requires a storage for the data per each player, separately from Authorization / Authentication microservice
- Necessary to track a player progress after each game and perform certain actions on this data 

### Benefits of using
- The data can be easily denormalized depends on your use cases
- The stored data can be used for analytics and tracking for a state of the game (balance, the most picked characters and so on)

### Entity relationship diagram
#### Statistic table
| Field name      | Field type | Constraints | Description                                |
|-----------------|------------|-------------|--------------------------------------------|
| id              | UUID       | PK          | A database unique object identifier (same as in Authorization / Authentication microservice) |
| games           | Integer    | NOT NULL    | Total number of games played                       |
| wins            | Integer    | NOT NULL    | Total number of games when the player won          |
| loses           | Integer    | NOT NULL    | Total number of games when the player was defeated |
| rating          | Integer    | NOT NULL    | An actual rating of the player                     |

Also the table can have an extra fields in database (for example, extra columns/documents in NoSQL or a nested JSON/HStore field in RDBMS) that can be used for providing for a detail information about the player.

### RabbitMQ queues
| Queue name                      | Exchange name                               | Usage                              | Returns                         |
|---------------------------------|---------------------------------------------|------------------------------------|---------------------------------|
| player-stats.statistic.init     | open-matchmaking.player-stats.statistic.init.fanout     | Initializes statistics from an empty state for the new player    | Statistics or a validation error |
| player-stats.statistic.retrieve | open-matchmaking.player-stats.statistic.retrieve.fanout | Returns the player statistics    | Statistics or a validation error |
| player-stats.statistic.update   | open-matchmaking.player-stats.statistic.update.fanout   | Updates the player statistics | Statistics or a validation error |
