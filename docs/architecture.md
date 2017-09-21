# Architecture

Summary
-------
- [**How it works**](https://github.com/OpenMatchmaking/documentation/blob/master/docs/architecture.md#how-it-works)
  - [**Without Auth/Auth microservice**](https://github.com/OpenMatchmaking/documentation/blob/master/docs/architecture.md#without-authauth-microservice)
  - [**With Auth/Auth microservice**](https://github.com/OpenMatchmaking/documentation/blob/master/docs/architecture.md#with-authauth-microservice)
- [**Protocol**](https://github.com/OpenMatchmaking/documentation/blob/master/docs/architecture.md#protocol)

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
4) Server adds a new asynchronous task in tasks pool (i.e. Celery with Python 3+) and after it, starts processing a new request.  
5) Doing business logic:  
  5.1. On of the existing worker is running an appropriate code that will do some useful work (building a leaderboard, searching a player/team).  
  5.2. If it will be necessary, this code could take same data from the storage which stores information about finished maches, players and so on. For instance, during activity of the player all data could be stored in NoSQL storage (like CouchDB, Mnesia, Redis, etc.) and any other data in some relational database (like PostgreSQL, MySQL, etc.).
6) When all required work were completed, finished task put the message with player ID into message queue (it could be RabbitMQ, ZeroMQ, etc.)
7) Because our backend server is subscribed on the messages with particular player ID, it just received this message from a message queue, extract the result from it, make some additional post-processing actions (if it will be necessary) and put the new message in one on message queues.
8) Because the reverse proxy is listening messages with the particular request_id, it extract the message from a message queue, when it will be appeared and prepare the response for the game client.
9) Send a response to the waiting game client with the data that was prepared on the previous step.
10) Game client is trying to connect to the prepared game server, entering into already created game lobby and waiting until other players will be connected.
11) After the game is prepared, game server started a new game with the connected users. The game server and game client just communicating with each other and sychronizing game states and do some additional work that required during the current game session.
12) When the game was finished, game server sends a request with information about the completed game in body to the redirected reverse proxy node, selected by the load balancer.
13) Proxy server like on the previous steps, wraps a request into a "message" and put it in one of available message queues. Also subscribing to getting a response from a some existing processing node. When the message will be recieved, will return it to a caller.
14) One of the appropriate servers which is could process it, takes the message from the message queue. Unwraps the message, and do some useful work. Puts the message into a message queue with the label, that data processing was started.
15) Matchmaking service received a request to save this part of data, and update/refresh it in the appropriate storage.

And so on... Each inner microservice of Open Matchmaking could do some work and could returns (or not) a response to the game client / game server if it will be necessary. It all depends on the particular case of using it and what necessary to do.

### With Auth/Auth microservice
This case will be good for the project when:
- In cloud each component has its own Authentication / Authorization layer. And Open Matchmaking platform should not be an exception.
- The project have a private cloud, but you can't get an access to add some external dependencies and necessary somehow provide a way to connect players together.
- Necessary to divide the project on the small parts, which can be developed by different developers and studios (like sharing of responsibility), but you need also a Authentication / Authorization layer.

<p align="center">
  <img src="https://github.com/OpenMatchmaking/documentation/blob/master/docs/images/architecture-with-auth.png"/>
</p>

Protocol
--------
This following JSON-based message protocol will be used for communications between different microservice nodes, like API Gateway and matchmaking microservice. The message envelop separated on the two parts:
- Header info
- Content

### Header info
- Request ID - A unique identifier per each incoming request. Should be specified for any message, that passed into/taken from a message queue.
- Microservice name - A unique microservice name which is used for understanding which service should process this request.
- Request URI - String which is used to identify a resource.
- Expires-At - Specified the time, when the message should be dropped from a message queue. Optional.
- Token - Unique token specified for each client. Required when Authentication / Authorization layer was enabled.

### Content
Contains data, that could be used for processing by one the existing microservices. If no data required, that left this field as an empty dictionary.

### Example of incoming request from a game client
```javascript
{
  "header_info": {
    "Request-ID": "6ed85c05-7302-402c-892c-1ae3f78ac355"
    "Microservice-Name": "matchmaking",
    "Request-URI": "/search-game/"
  }
  "content": {
    "Player-ID": "0146563d-0f45-4062-90a7-b13a583defad",
    "Game-Mode": "team-deathmatch"
  }
}
```
