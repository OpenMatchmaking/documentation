# Componets

Here you can find a list of components which are сompose together the Open Matchmaking platform.

Summary
-------
- [**Reverse Proxy**](https://github.com/OpenMatchmaking/documentation/blob/master/docs/components.md#reverse-proxy)

Reverse proxy
------------
### Why 
- Keep internal parts are hidden for a typical user, so that he still thinking that only works via WebSockets.
- Isolate users from an access to a message broker.
- Use it as an additional layer when necessary to use authentication / authorization layer.

### How it works
What is reverse proxy?
> Is a type of proxy server, which is retrieving resources on behalf of a client from one or more servers. Each of those resources are returned to the waiting client so that it looks like a they was prepared by the web server itself. Moreover, each reverse proxy is an intermediate node for its associated servers to be contacted by any client.  

Now, when you was get an idea what is it, let's see on the next scheme:

<p align="center">
  <img src="https://github.com/OpenMatchmaking/documentation/blob/master/docs/images/reverse-proxy.png"/>
</p>

1) Client connecting with a reverse proxy via some protocol (for instance, HTTP or WebSockets). And sends some part of data to it. If necessary to check credentials / tokens / etc. the reverse proxy also can do this work.
2) When all pre-processing steps are сompleted successfully, the reverse proxy should make "protocol transformations", which is means is get an incoming request, extract data and prepare it as an another request to a message broker (it could be AMQP, Kafka, etc.). If required additional actions, like subscribe on something, then the reverse proxy can make it too. If will be useful when necessary to return certain response.
3) One of existing microservices listening on messages with the particular headers / topic, and extract when it comes, do some work and put the response backward at the message broker.
4) If certain API endpoint is requiring to prepare response and return it to the client, then reverse proxy extract the message from a message broker, when prepared message was put into it by microservice.
5) The reverse proxy is doing "reverse protocol transformations", so that client will receive data in the same format (in which it was sent), convenient for using further.

### Benefits of using
- Reverse proxy don't care about the processing requests, because it need only apply transformations for an incoming message, put into the queue and retrieve the response from one of microservices, which is work with message broker.
- Client can send request for processing for more than one microservice with one request (and sometimes it could be really useful).
- All entire structure of your project is hidden. Only one an entry point.
- Slightly adjusting mappings between incoming requests and queues gives you a lot of ways to solve your task. 
- Each inner part of your system don't need to connect to reverse proxy. All of them are working with message broker, and don't care about how the requests are coming into the queues.  
