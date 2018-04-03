# Protocol

The communication process between different microservices will be organized with a message broker via using JSON-based messages in its body. Each existing microservice should retrieve and return data as JSON objects, so that it will be easy to write, maintain and debug the potential issues further.

## Messages for clients
Each incoming message should contain a couple of fields:
- `url` - Represent a URL to a specific resource. Must be specified only request message. Required.
- `content` - Represents a message data for using by the certain microservice. Can be specified in request and/or response messages. Optional.
- `token` - Represents a JSON Web Token as a string. Must be specified when the Authentication / Authorization layer for reverse proxy was enabled. Can be specified only for request message. Optional.
- `event-name` - Used for determining from which endpoint the response came. When isn't interested in response, set it to `null`. Optional.

### Example of a request from a client to a microservice
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

### Example of a response from a microservice to a client 
```javascript
{
  "content": {
    "Server-IP": "127.0.0.1:5000",
    "Access-Token": "Secret token"
    "Team": "red"
  },
  "event-name": "find-opponents"
}
```

## Errors
In the case of errors, the server must returns a list of error, so that the client could process the result response and somehow render or display the result of operation on the screen. The response body contains two required fields:
- `errors` - List of objects, where each object is represented is specified in `{"title": "description"}` format. The desciption of the error can be a string, list or dictionary type.
- `event-name` - A string identifier for a response, with the help of which it is possible to understand from which microservice the response will return.

### Example
```javascript
{
  "errors": [{"Routing error": "The requested resource does not exist."}],
  "event-name": "find-opponents"
}
```

## Messages for microservices
Each prepared message for communicating betwee microservices contains two parts:
- Headers (Required)
- Content (Required)

### Headers
Custom RabbitMQ headers:
- `microservice_name` - A unique microservice name which is used for understanding which service should process this request.
- `request_url` - The original URL as a string which is used to identify a resource.
- `permissions` - A list of permissions to resources splitted with the comma. Each permission must be specified in `{microservice}.{resource}.{action}` format.
- `request_source` - Specifies the source from which request was coming. Represented as the value of the `RequestSource` enum with the following values:
  - `client` (or `0`) - Request comes from outside of Open Matchmaking platform. This value is set by reverse proxy, per each request from the client by default.
  - `internal` (or `1`) - Request comes from an internal microservice of Open Matchmaking platform. Must be set by internal microservice-sender by default.
- `user_id` - A unique UUID identifier, generated and stored in Auth / Auth microservice per each user.

Default RabbitMQ headers:
- `content_type` - Specifies the data type in the message body. Default is `application/json`.
- `delivery_mode` - Specified the delivery mode for the message. Set by reverse proxy to the `2` and means that the message must be `persisent` in terms of RabbitMQ.
- `reply_to` - A unique UUID4 identifier to an exclusive response client queue, to which necessary deliver a final response.
- `correlation_id` - A string identifier for a response, with the help of which it is possible to understand from which microservice the response will return.

### Content
Contains an information, that will be used for processing by one the existing microservices. If no data required, that left this field as an empty string.
