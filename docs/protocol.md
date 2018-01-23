# Protocol

The communication process between different microservices will be organized with a message broker with JSON-based messages in its body. Each existing microservice should retrieve and return data as JSON objects, so that it will be easy to write, maintain and debug the potential issues further.  

Each message is split onto three parts:
- Headers (Required)
- Content (Required)
- Token (Optional)

### Headers
- Request ID - A unique identifier per each incoming request. Should be specified for any message, that passed into/taken from a message queue. Can be used as a response topic/queue name for a certain client.
- Microservice name - A unique microservice name which is used for understanding which service should process this request.
- Request URI - A string which is used to identify a resource.
- Permissions - A list of permissions to resources. Optional.
- Event-Name - A string identifier for a response, with the help of which it is possible to understand from which microservice the response will return.

### Content
Contains an information, that will be used for processing by one the existing microservices. If no data required, that left this field as an empty dictionary.

### Token
Represent a unique JSON Web Token per each client for getting an access to microservices functionality. Must be specified when the Authentication / Authorization layer for reverse proxy was enabled.

### Example of a request
```javascript
{
  "headers": {
    "Request-ID": "6ed85c05-7302-402c-892c-1ae3f78ac355"
    "Microservice-Name": "matchmaking",
    "Request-URI": "/search-game/",
    "Permissions": "matchmaking.search.get; leaderboard.potg.get",
    "Event-Name": "get-opponents-event"
  },
  "content": {
    "Player-ID": "0146563d-0f45-4062-90a7-b13a583defad",
    "Game-Mode": "team-deathmatch"
  },
  "token": "a unique token"
}
```
