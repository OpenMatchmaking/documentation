# Default exchanges and queues

Here you can find the default RabbitMQ exchanges and queues which are available for any existing microservice in Open Matchmaking platform. 

Any other exchanges must use its own exchanges (which most preferably) and follow the `open-matchmaking.{microservice}.{resource}.{operation_type}.{exchange_type}` format, where:
- `{microservice}` is the microservice name.
- `{resource}` is a resource / database table name.
- `{operation_type}` operation under the resource. For example it can be:
  - `retrieve` for receiving a detail of some object.
  - `create` for creating a new object. 
  - `update` for updating the object.
  - `delete` for deleting the object.
- `{exchange_type}` as the name of the one of possible AMQP exchange types:
  - `direct` for messages which are delivering to the certain queue based on the routing key.  
  - `fanout` for messages which are delivering to all queues that were bounded to the specified exchange.
  - `topic` for messages which are delivering to certain queues based on matching between a routing key and the pattern that was used to bind a queue to an exchange.
  - `header` for messages which are delivering to the queues via using message headers.

#### Exchanges
| Exchange name                      | Exchange type | Options                                        |
|------------------------------------|---------------|------------------------------------------------| 
| open-matchmaking.direct            | direct        | durable=True, passive=False, auto_delete=False |
| open-matchmaking.fanout            | fanout        | durable=True, passive=False, auto_delete=False |
| open-matchmaking.headers           | headers       | durable=True, passive=False, auto_delete=False |
| open-matchmaking.topic             | topic         | durable=True, passive=False, auto_delete=False |
| open-matchmaking.responses.direct  | direct        | durable=True, passive=False, auto_delete=False |
| open-matchmaking.responses.fanout  | fanout        | durable=True, passive=False, auto_delete=False |
| open-matchmaking.responses.headers | headers       | durable=True, passive=False, auto_delete=False |
| open-matchmaking.responses.topic   | topic         | durable=True, passive=False, auto_delete=False |
