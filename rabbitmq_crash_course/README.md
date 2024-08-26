# Notes from this [crash course](https://www.youtube.com/watch?v=Cie5v59mrTg)

- Publisher has a stateful two-way connection between itself and RabbitMQ server. It uses TCP with Advanced Message Queuing Protocol (in the application layer?) on top. 
- Consumer also has a stateful two-way TCP connection between itself and the RabbitMQ server, with the same AMQP protocol.  
- The RabbitMQ server pushes messages to the consumer when they’re ready. 
- Both the consumer and publisher can have multiple logical connections inside the same TCP connection via multiplexing.  
- RabbitMQ has an exchange queue in the backend, exposing only an “exchange” that the consumer deals with. The publisher publishes data, this data gets into the queue, and then pushed to the consumer from the queue. The publisher and consumer are not aware of the queue. 
- To run RabbitMQ docker container, run this command:  `docker run --name rabbitmq -p 5672:5672 rabbitmq`. The first part is the port exposed from your machine. The second port is the default RabbitMQ port, which must be 5672.
  
