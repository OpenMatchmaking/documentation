# Matchmaking

Distributing tasks for a search
-------------------------------
For solving this complex task, especially with using microservices, we should make it asynchronous, because don't know actually how much time it takes. In theory it could be infinitely, but on practice it doesn't occur, because we could stop search after a long period of time or re-run a search but with slightly updated parameters.

The basic idea for distributing task for searching is split up players by some criterias on small groups. For example, by a current rating or range of it. The more groups we create, then more balanced will be games between the players. But, of cource, it's necessary to keep in mind that their should not be too much relatively to overall amount of active players. 

How much groups of players should be? It mainly depends on the game, to which you are going to add the competitive constituent. For example, for MOBA games we could split up the people on groups with 500 rating points difference between the groups (alike in Ovewatch and League of Legends titles):
  - 1000 - 1500 => Bronze
  - 1500 - 2000 => Silver
  - 2000 - 2500 => Gold
  - 2500 - 3000 => Platinum
  - 3000 - 3500 => Diamond
  - 3500 - 4000 => Semi-Pro
  - 4000 - 4500 => Professional

Then, after creating groups, necessary to figure out, how to processing requests from a clients that would like to play in game with other players. Well, the simpler and more understandable the solution, the better the result we will get. Look at the following picture, how it can be potentially solved:

<p align="center">
  <img src="images/matchmaking/distributing-search-tasks.png"/>
</p>

The demonstrated schema is pretty straightforward. For getting an idea how it should work, we will consider a case for searching a game for a diamond player (for any other league it will work by analogy): 
1) The client sends the message for processing to a generic queue. For server response the client will create, register and subscribe for updates a response queue.
2) The "message consumers" will extract a message from it and transmit it into certain queue for further processing, which is depends on the elo, specified in the incoming message. 
3) Worker is trying to find a suitable player for a new game:  
  3.1. Because each worker is linked to the specific queue, it will extract message in sequence and try to analyze it. If the player, the information about which was specified in the message body, is according to the matchmaking algorithm, then the selected player will be saved in the memory of the worker and the extracted message deleted. This process is repeating until the worker have not enough players to fill the game lobby. And when it will completed, a list of players will be transferred as a message to the next queue.  
  3.2. Otherwise the message will be published the special queue, created for requeuing players into generic queue.  
  3.3. Special type of workers are re-publishing messages to the generic queue, which are coming from the publishing node from the 3.2 step.
4) A worker is extracting the published message on the previous step, and creating a new game server (or choosing one from the already existing). After that it will broadcast the server IP-address, port and connection credentials to each player mentioned in the list via particular response queues, that were specified by clients in the first step.
5) Each client is getting the response from response queue, connecting to the game lobby.

So, as you can see the logic here is pretty simple. Why it was developed this way? The answer is obvious: to get the as much as possible efficient and scalable servers for processing incoming requests, so that each part can be scaled by your needs and expectations for your own game.

Saving the worker state
-----------------------
Because each worker is using its own state for saving a list of the grouped players with a similar rating or a skill, we must be ensured that even in failure of the processing node the state will be reused and the data won't be lost. In the case when we aren't using any storage for saving the worker state, it is really important to understand that under the highload the node in the cluster can start and shutdown constantly what is leading to losing all processed data.

For dealing with this particular issue each existing worker will be store its state in the external storage (for example, it can be a RDBMS or a NoSQL solution), which is must provide high availability and fault tolerance features. In general the using of the "shared state" by workers is pretty straightforward:

1. Is initializing by the following schema:  
  1.1. It's trying to get one of the possible states with the data from the external storage. If it wasn't found, then the worker starts with an empty state.   
  1.2. Starts the periodic task in background for saving the state in the external storage as is each 5 seconds.  
2. Does a required amount of work and saving the result in the inner state as the part of worker after each successful processing.

This solution brings to us the next advantages:

- Easier to build the system with concurrency, because each worker can switch between different states atomically and replace the failured node when it happens.  
- Reduces the waiting time per each player, because the worker can return to the partially prepared group of players more often. 
- The cluster becomes more stable and less prone to data loss.

But, of course, it have some disadvantages as well:

- Harder to implement the desired behaviour with concurrency support.
- Increases a maintaining cost and required time for preparing a working environment.
- Adds additional network delays during the work.

Searching games with considering player's location
--------------------------------------------------
Some people can be curious about how to search a team for players that have different locations (e.g. on continents). So, for it a developer have three different ways to handle it, just pick the most suitable solution to you:
1) Split up players audience by regions in which they are playing  
  <br>Instead of letting to people to play from the different location in the one game, you could split up them by regions (like onto North America, Europe, Asia) for getting better game experience and team balance. To achieve this, you need to provide users an opportunity to register in each separate region. Also using this solution, you do not need to configure the project for each region as a whole, because the codebase is identical, but divided only into service regions (e.g. via cloud services).

2) Add a new stage of processing for [Matchmaking microservice](components/matchmaking-microservice.md)  
  - In the case of using this approach, everything you need is to do is add another processing stage, for example, after a generic queue. When using this solution, you get some flexibility in processing requests from users, because you have a complete control over the data with a group of players and could apply additional filtering and balancing rules when the player has too high latency.   

<p align="center">
  <img src="images/matchmaking/with-location-filter-stage.png"/>
</p>

  - On the other hand, the project could use several different queues, each of which only serves a specific region for players. For example, players from North America could be placed in the diamond queue only with similar players from the same location, where is from Europe only with each other. This approach also has the right to use, although it requires additional efforts on support and development.

<p align="center">
  <img src="images/matchmaking/separate-region-queues.png"/>
</p>
  
3) Extend the functionality of [Strategist microservice](components/strategist-microservice.md)   
  <br>In the case when you can provide information about a player (e.g. his location and average delay), this solution can be quite good, because you're communicating only with the certain microservice, that groups up players in one team (or maybe even in the game in general) with close location. At the same time, you will not need to change the logic of other components of the system - all changes occur in one microservice that deals only with building fair matches.

Strategies of a searching players
---------------------------------
In general it is a complex task to solving that doesn't have a single solution, because you will need to analize a lot of active players, so that each player would be picked and placed in equal conditions with others players. All players which are playing a match on the game server must have close same game skills or levels, so that each player (or a team) have the close chances for a win and a defeat.

### Elo-based
Description: This method of calculating for games is presuming that each player have a calculating rating, represented by a number which increases or decreases depending on the outcome of games between other players. After each game, the winner takes the points from the losing side. Its amount depends from a difference in rating between the played gamers. The simplest implementation could relies onto classical [Elo rating](https://en.wikipedia.org/wiki/Elo_rating_system) ideas and calculations.  

Example of games: League of Legends (1-3 seasons of comptetive gaming)

Pros:
  - A good fit for a simple type of games and oriented mostly for 1v1 matches
  - Objectively reflects the strength of the player in actual the moment
  - Easy to implement an algorithm for solo matchmaking and matches in groups

Cons:
  - Rating is pretty unstable for the first matches when comes a new player into the game
  - Not suitable for games that emphasize a team play

### Average rating
Description: Using an average rating is presuming that the players are playing in a team-based game, so that each team is always contains a couple of players. For each team is calculating an average of rating for the group and used in searching the enemy teams that have a close rating as well (but not above a certain threshold). After a game each player from winners team receive a points from the defeated team, that were calculated with a rule or a formula.  

Example of games: Overwatch

Pros:
  - Works good for team-based games, when each player playing without a group
  - Stable and enjoyable games even for new players
  - Simple to implement for the cases when players are playing in the group

Cons:
  - The greater the difference in rating among the players in the group, the worse for the players without the group but in the same team
  - Hard to find an optimal balance by skill for a few groups of players in different teams

### Peformance-based selection
Description: This search algorithm based on the idea that a place in game lobby for each player will be found based on his actual statistics data and performance (for example, the most played game role and character, an average dealt damage, how often he dies and so on). After a win the winners side receivies the points from the defeated side, that calculated with a rule or a formula as well.  

Example of games: Heroes of the Storm

Pros:
  - Enough accurate than other methods and works good for different types of games and modes
  - Knows how to balance players with variant skill level, even if they're in a one group
  - Can be adapted to the certain case, so that you will get a good balance for players

Cons:
  - Necessary to have a good "knowledge database" for analytics, so that it will be easy to figure out what the type of player in front of you and how to balance him relatively to other players and teams
  - Hard to find generic characteristics for players. Need some time and a bunch of iteration for searching the most valuable characteristics
  - Should be constantly adapted to the gameplay changes
