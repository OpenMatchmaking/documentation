# Protocol

The communication process between different microservices will be organized with a message broker via using JSON-based messages in its body. Each existing microservice should retrieve and return data as JSON objects, so that it will be easy to write, maintain and debug the potential issues further.

## Messages for clients
Each incoming message should contain a couple of fields:
- `url` - Represent a URL to a specific resource. Required.
- `content` - Represents a message data for using by the certain microservice. Optional.
- `token` - Represents a JSON Web Token as a string. Must be specified when the Authentication / Authorization layer for reverse proxy was enabled. Optional.
- `event-name` - Used for determining from which endpoint the response came. When isn't interested in response, set it to `null`. Optional.

### Example of a incoming message
```javascript
{
  "url": "api/v1/matchmaking/search"
  "content": {
    "Player-ID": "0146563d-0f45-4062-90a7-b13a583defad",
    "Game-Mode": "team-deathmatch"
  },
  "token": "a unique token"
  "event-name": "find-opponents"
}
```

## Messages for microservices
Each prepared message for communicating betwee microservices contains two parts:
- Headers (Required)
- Content (Required)

### Headers
- Microservice name - A unique microservice name which is used for understanding which service should process this request.
- Request URI - A string which is used to identify a resource.
- Permissions - A list of permissions to resources. Optional.
- Event-Name - A string identifier for a response, with the help of which it is possible to understand from which microservice the response will return.
- Token - A unique JSON Web Token per each client for getting an access to microservices functionality. Must be specified when the Authentication / Authorization layer for reverse proxy was enabled.

### Content
Contains an information, that will be used for processing by one the existing microservices. If no data required, that left this field as an empty string.
