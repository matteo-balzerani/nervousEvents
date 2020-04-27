
# NervousEvents MVP 

This MVP wants to show one approach to the problem showed and a flexible solution is presented and builded.

The solution is composed by 5 different micro services, 1 publish-subscribe system (kafka) and 1 database (mongoDB)
The follow image can explain, from a general perspective, how those services are related and how the message-flow is managed.

## List of services
1. ### [HandlerMessageService](https://github.com/matteo-balzerani/HandlerMessageService)
This service is used as an entry point. 
A user/publisher can use a rest api interface to send a message to the system choosing a topic. 
With a simple Post request to [/api/messages/publish/{topic}](http://localhost:8088/swagger-ui.html#/operations/messages-controller/publishUsingPOST) a message is created and sent to the {topic} topic. If {topic} doesn't exist, it will be created.

In the complete solution this service will be used to accept different type of events and redirect those to the publish-subscribe tool.
Also a security layer can be builded on top of this service

2. ### [Apache Kafka](https://kafka.apache.org/) (using Zookeeper)
This service is configured in order to manage different topics, handle the messages and serve those to the subscribers. 
If the subscribers are systems they can consume the messages related to specific topic(s) using the standard configuration from Kafka. In this case more configuration is needed but it really depends on requirements.
Also if more actions are needed related to the messages, those can be configured.
This MVP provide (for default) a single consumer totally managed. 

The simple configuration used in this MVP is enough to configure a minimum kafka/zookeeper couple usable for the basic usage.
In the complete solution this layer can be more and more complex. Also a cloud based solution can be used.

3. ### [ForwardMessageService](https://github.com/matteo-balzerani/ForwardMessageService)

This service is used as a consumer for all the messages pushed in Kafka.
A single consumer is listening on all the topics and every new topic created will be included.
When a message is consumed a HttpClient try to send it to all the subscriber registered. If a subscriber cannot receive the message, the system will save this message in MongoDB.
The list of all subscriber (and the topics they are interested in) is stored in the same mongoDB and it is managed by another service.
The consumer is not optimized for hundreds of topics or thousand of messages per seconds. However the optimization is tightly related to the business logic, the scope of any topics and it is definitely out of the scope of this MVP. 

4. ### [MongoDB](https://www.mongodb.com/)
this database is used to store 2 different collections:
* Subscriber : handle all the subscribers to a topic. 
* Messages : every no-consumed message is store with the not available subscriber related.

5. ### [ReaderFailedMessageService](https://github.com/matteo-balzerani/ReaderFailedMessageService)

This service is composed by 2 different scheduler. Both have the same behaviour but a different rate. 
Every scheduler reads the unconsumed messages from MongoDB and try to resend it to the specified subscriber. If the message is sent correctly the record is deleted from the collection, otherwise not.
The first scheduler is configured to run every 10 second and takes care about all the messages not consumed in the previous five minutes.
The second scheduler is configured to run every 30 seconds and takes care about all the messages not consumed in between the previous 5 and 10 minutes.
Every scheduler, rate, time, can be configured, but in this MVP they are fixed.

6. ### [RestApiInterface](https://github.com/matteo-balzerani/RestApiInterface)

This service is a simple web rest api service used for two different   purposes: 
* Manage Subscriber: using a post request a subscriber can provide some information needed to the system in order to send him the messages related to a specific topic
* Allow the subscribers to request messages from a specific topic. using a get request a subscriber can requests all the unread messages from a topic.

In this service many other actions can be configured but for the scope of this MVP the 2 already discussed should be enough.

7. ### [mockedSubscriber](https://github.com/matteo-balzerani/mockedSubscriber)

the simpler python web service ever done. 
it used to mock an external subscriber and it only accept POST request at /mock url. 
Also this service use a random number to decide it the response will be 201 (80%) or 400 (20%). In this way a broken subscriber can be simulated.

## Aggregation
All the service are dockerized and in this MVP they can be used together using the docker-compose file in this repo.
A better orchestrator can be chosen (i.e. K8s) but in order to create this proof-of-content the compose is enough. Also the orchestration is (with the configurations) a topic coupled to the business and the scope of the all the system

## Tech note:
* All the services are builded as a SpringBoot application. this allow a fast development and a simple basic configuration. 
* Java 8 is used in all the services.
* Security is not used in all the system (MVP don't need it)
* Tests are not yet implemented due to fast development and also because they are not mandatory from a proof-of-content perspective.
* The mockedSubscriber is builded as a flask service in python3
