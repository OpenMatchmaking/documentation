# Architecture

Summary
-------
- [**How it works**](https://github.com/OpenMatchmaking/documentation/tree/master/docs#how-it-works)
- [**Protocol**]()

How it works
------------
The matchmaking services are working by the following schema:

<p align="center">
  <img src="https://github.com/OpenMatchmaking/documentation/blob/master/docs/images/Architecture.png"/>
</p>

As you can see at this image it contains three different components:
1. **Game client**. Represents a certain "unified shell" for each active user, which is connected to the main game server and exchanges data with it if necessary.
2. **Game server**. Represents an existing node in a cluster, which transmits data about its internal state of the particular game and allow its connected clients to maintain their own accurate version of the game world for display to players. Also could processing/receiving each player's input, messages and so on.
3. **Open Matchmaking platform**. That is the core component of our platform, to what we are aiming for. It do everything that related to the matchmaking and that is expected from this service: 
    - Searching an opponent with similar skills (or a suitable team, if it is a competitive game that focus on the teamplay) for each player, so that the chances will be close to the 50/50 percents of win/lose. And it is bringing a lot of opportunites to improve its skills and fun for everyone in each match.
    - Providing a lot different strategies (or matchmaking algorightms), that could be used for different game modes and genres at the same time.
    - Collecting some statistics, that could be used for further data processing, like creating the leaderboards, plays of the day and etcetera after ending each match.

The next part about we will b talk its how entire components works and communicating with each other. On the high level abstraction it works pretty simple:
1) The game client sends a request to an API gateway server to find a new match for a player. In request body we should specify the player ID, game mode and a used search strategy at least (but we could append something else, when it will be necessary and our server could use it information).
2) After receiving a request from a player, API Gateway transmit it to an appropriate server which is could process it. For example, one of the registered server in API Gateway can process only matchmaking requests, another only building a user profiles, and so on.
3) The processing node receives the request, and before its processing do subscribing on the messages with the player ID, that was specified in request.   
  **NOTE:** This step should be done only once when the player runned a game client, logged into system and runned a searching in the first time.
4) Server add a new asynchronous task in tasks pool (i.e. Celery with Python 3+) and after it, starting processing a new request.  
5) Doing business logic:  
  5.1. On of the exissting worker is running an appropriate code that will do some useful work (building a leaderboard, searching a player/team).  
  5.2. If it will be necessary, this code could take same data from the storage which stores information about finished maches, players and so on. For instance, during activity of the player all data could be stored in NoSQL storage (like CouchDB, Mnesia, Redis, etc.) and any other data in some relational database (like PostgreSQL, MySQL, etc.).
6) When all required work was completed, finished task put the message with player ID into message queue (it could be RabbitMQ, ZeroMQ, etc.)
7) Because our web server is subscribed on the messages with particular player ID, it just received this message and extract the result from it.
8) Return the final response to the API gateway.
9) Send a response to the waiting game client with the data that was extracted on the previous step.
10) Game client is trying to connect to the prepared game server, entering into already created game lobby and waiting until other players will be connected.
11) After the game is prepared, game server started a new game with the connected users. The game server and game client just communicating with each other and sychronizing game states and do some additional work that required during the current game session.
12) When the game was finished, game client send a request with information about the completed game in body to the API Gateway.
13) API Gateway like on the previous steps, transmits a request to an appropriate service, that could do this work.
14) Matchmaking service received a request to save this part of data, and update/refresh it in the appropriate storage.
