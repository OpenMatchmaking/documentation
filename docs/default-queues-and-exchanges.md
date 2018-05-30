# Default exchanges and queues

Here you can find the default RabbitMQ exchanges and queues which are available for any existing microservice in Open Matchmaking platform. 

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
