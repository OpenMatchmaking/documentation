# Architecture

Summary
-------
- [**How it works**](https://github.com/OpenMatchmaking/documentation/blob/master/docs/architecture.md#how-it-works)
  - [**Without Auth/Auth microservice**](https://github.com/OpenMatchmaking/documentation/blob/master/docs/architecture.md#without-authauth-microservice)
  - [**With Auth/Auth microservice**](https://github.com/OpenMatchmaking/documentation/blob/master/docs/architecture.md#with-authauth-microservice)
- [**Protocol**](https://github.com/OpenMatchmaking/documentation/blob/master/docs/protocol.md)

How it works
------------
Before you will dive into details, let's mention components, which are used during the architecture consideration:
1. **Game client**. Represents a certain "unified shell" for each active user, which is connected to the main game server and exchanges data with it if necessary.
2. **Game server**. Represents an existing node in a cluster, which transmits data about its internal state of the particular game and allow its connected clients to maintain their own accurate version of the game world for display to players. Also could processing/receiving each player's input, messages and so on.
3. **Open Matchmaking platform**. That is the core component of our platform, to what we are aiming for. It do everything that related to the matchmaking and that is expected from this service: 
    - Searching an opponent with similar skills (or a suitable team, if it is a competitive game that focus on the teamplay) for each player, so that the chances will be close to the 50/50 percents of win/lose. And it is bringing a lot of opportunites to improve its skills and fun for everyone in each match.
    - Providing a lot different strategies (or matchmaking algorightms), that could be used for different game modes and genres at the same time.
    - Collecting some statistics, that could be used for further data processing, like creating the leaderboards, plays of the day and etcetera after ending each match.

Now let's move on to the two diffent cases of usage Open Matchmaking platform for your own game. 

### Without Auth/Auth microservice
This use case will be a good fit when:
- Have a private cloud and want to use Open Matchmaking platform as a part of existing infrastructure, which is also has implemented Authentication / Autorization layer. If you're going this way, then necessary to keep in mind, that you've made additional check (for instance, check permissions to get an access to matchmaking part).
- No need Auth / Auth part (but need to be care, if you're doing so).

Nonetheless, in this case the matchmaking services are working by the following schema:

<p align="center">
  <img src="https://github.com/OpenMatchmaking/documentation/blob/master/docs/images/architecture-no-auth.png"/>
</p>

1) The game client sends a request to find a new match for a player to a load balancer, which is redirecting user to an existing reverse proxy server. In request body we should specify the player ID, game mode and a used search strategy at least (but we could append something else, when it will be necessary and our server could use it information).
2) After receiving a request from a player, reverse proxy will wrap this request into a "message" and put it in one of available message queues, which are listening by all existing matchmaking microservices. For getting a response from a one of processing nodes, reverse proxy just listening for a message in a mailbox with the appropriate request_id, which was set by it.
3) One of the appropriate servers which is could process it, takes the message from the message queue. The processing node receives the request, and before its processing do subscribing on the messages with the player ID, that was specified in request.   
**NOTE:** This step should be done only once when the player runned a game client, logged into system and runned a searching in the first time.
4) Server adds a new asynchronous task in tasks pool (i.e. Celery with Python 3+) or delegate a task to an actor on the server (Elixir/Erlang actors, Skala with Akka framework and so on) and after it, starts processing a new request.  
5) Doing business logic:  
  5.1. On of the existing worker is running an appropriate code that will do some useful work (building a leaderboard, searching a player/team).  
  5.2. If it will be necessary, this code could take same data from the storage which stores information about finished maches, players and so on. For instance, during activity of the player all data could be stored in NoSQL storage (like CouchDB, Mnesia, Redis, etc.) and any other data in some relational database (like PostgreSQL, MySQL, etc.).
6) When all required work were completed, finished task put the message with player ID into message queue (it could be RabbitMQ, ZeroMQ, etc.).
7) Because our backend server is subscribed on the messages with particular player ID, it just received this message from a message queue, extract the result from it, make some additional post-processing actions (if it will be necessary) and put the new message in one on message queues.
8) Because the reverse proxy is listening messages with the particular request_id, it extract the message from a message queue, when it will be appeared and prepare the response for the game client.
9) Send a response to the waiting game client with the data that was prepared on the previous step.
10) Game client is trying to connect to the prepared game server, entering into already created game lobby and waiting until other players will be connected.
11) After the game is prepared, game server started a new game with the connected users. The game server and game client just communicating with each other and sychronizing game states and do some additional work that required during the current game session.
12) When the game was finished, game server sends a request with information about the completed game in body to the redirected reverse proxy node, selected by the load balancer.
13) Reverse proxy like on the previous steps, wraps a request into a "message" and put it in one of available message queues. Also subscribing to getting a response from a some existing processing node. When the message will be recieved, will return it to a caller.
14) One of the appropriate servers which is could process it, takes the message from the message queue. Unwraps the message, and do some useful work. Puts the message into a message queue with the label, that data processing was started.
15) Matchmaking service received a request to save this part of data, and update/refresh it in the appropriate storage.

And so on... Each inner microservice of Open Matchmaking could do some work and could returns (or not) a response to the game client / game server if it will be necessary. It all depends on the particular case of using it and what necessary to do.

### With Auth/Auth microservice
This case will be good for the project when:
- In cloud each component has its own Authentication / Authorization layer. And Open Matchmaking platform should not be an exception.
- The project have a private cloud, but you can't get an access to add some external dependencies and necessary somehow provide a way to connect players together.
- Necessary to divide the project on the small parts, which can be developed by different developers and studios (like sharing of responsibility), but you need also a Authentication / Authorization layer.

It works almost in the same way as [without the Authentication / Authorization microservice](https://github.com/OpenMatchmaking/documentation/blob/master/docs/architecture.md#without-authauth-microservice), except that we have here an addional layer, that let you to get an access to Open Matchmaking platform.

<p align="center">
  <img src="https://github.com/OpenMatchmaking/documentation/blob/master/docs/images/architecture-with-auth.png"/>
</p>

1) Client sends request to the Auth/Auth microservice and provides some required data (login/password, etc.) 
2) An existing node takes the request, and started to processing it in the few steps:  
  2.1. Checks the credentials for a user. If they aren invalid - returns an error, otherwise going further.  
  2.2. Generating a new token for the client. If before processing the request, token already exists, then return it.  
  2.3. Selecting one of reverse proxy IPs from a table in a database with some algorithm (which is to pick it with taking into account the active load), to which our game client will be connected.  
  2.4. Prepare the response for a client, that contains reverse proxy IP and access token.  
3) Auth/Auth microservice returns the generated response to the game client.
4) Game client is trying to establish a connection with one of reverse proxy instances, via getting an access to it through a load balancer that redirect to one of them on the first attempt of an access. If connection attempt was failed (reverse proxy node was shutdown or somehow disconnected from a cluster), then game client should pass 1-3 steps once again. Otherwise going further.
5) Reverse proxy accepts connection with a client and checking the access token, that was specified by the connected client in request. If it invalid or expired, then returns an error. Otherwise increases the counter for an active users on the proxy server and updates the last time of accessing to the node, which are storing in the same database for Auth/Auth microservice.
6) Reverse proxy will wrap this request into a "message" and put it in one of available message queues, which are listening by all existing matchmaking microservices. For getting a response from a one of processing nodes, reverse proxy just listening for a message in a mailbox with the appropriate request_id, which was set by it.
7) One of the appropriate servers which is could process it, takes the message from the message queue. The processing node receives the request, and before its processing do subscribing on the messages with the player ID, that was specified in request.   
**NOTE:** This step should be done only once when the player runned a game client, logged into system and runned a searching in the first time.
4) Server adds a new asynchronous task in tasks pool (i.e. Celery with Python 3+) or delegate a task to an actor on the server (Elixir/Erlang actors, Skala with Akka framework and so on) and after it, starts processing a new request.
9) Doing business logic:  
  9.1. On of the existing worker is running an appropriate code that will do some useful work (building a leaderboard, searching a player/team).  
  9.2. If it will be necessary, this code could take same data from the storage which stores information about finished maches, players and so on. For instance, during activity of the player all data could be stored in NoSQL storage (like CouchDB, Mnesia, Redis, etc.) and any other data in some relational database (like PostgreSQL, MySQL, etc.).
10) When all required work were completed, finished task put the message with player ID into message queue (it could be RabbitMQ, ZeroMQ, etc.).
11) Because our backend server is subscribed on the messages with particular player ID, it just received this message from a message queue, extract the result from it, make some additional post-processing actions (if it will be necessary) and put the new message in one on message queues.
12) Because the reverse proxy is listening messages with the particular request_id, it extract the message from a message queue, when it will be appeared and prepare the response for the game client.
13) Send a response to the waiting game client with the data that was prepared on the previous step.
14) Game client is trying to connect to the prepared game server, entering into already created game lobby and waiting until other players will be connected.
15) After the game is prepared, game server started a new game with the connected users. The game server and game client just communicating with each other and sychronizing game states and do some additional work that required during the current game session.  
16) When the game was finished, game server sends a request with information about the completed game in body to one of existing reverse proxy node. Of course and for this case as well, before passing a data to the appropriate reverse proxy instance, it will necessary to refer to the load balancer that will send a client to a separate processing node.   
**NOTE:** In order to avoid cluttering the scheme the part about communicating with Auth/Auth part and checking access token is hidden. But game server should do the same things as the game client (get the reverse proxy IP with token and check the generated token on the reverse proxy side later).
17) Reverse proxy like on the previous steps, wraps a request into a "message" and put it in one of available message queues. Also subscribing to getting a response from a some existing processing node. When the message will be recieved, will return it to a caller.
18) One of the appropriate servers which is could process it, takes the message from the message queue. Unwraps the message, and do some useful work. Puts the message into a message queue with the label, that data processing was started.
19) Matchmaking service received a request to save this part of data, and update/refresh it in the appropriate storage.
