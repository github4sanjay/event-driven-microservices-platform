Microservices have been around for some time, microservices architecture decouples large, complex systems into simple, independent services. Our subject today is one of the most awesome  - Event driven microservices architecture. Event-driven architecture is a methodology used to produce, handle events and implement applications where events transmit among decoupled software components and services.

Here's our architecture visualization.

![](https://media-exp2.licdn.com/dms/image/C4E12AQE4EtKrYNDqkw/article-inline_image-shrink_1500_2232/0?e=1584576000&v=beta&t=GtSJnXu4v41l9ZBKm_briyJFMPluPrC2z6E7Fq3QIjI)

Let's imagine that we are building an appointment scheduling application that creates account for a customer, sends notification and schedules store appointments.

We have four decoupled services: Store, Account, Notification and Appointment. All of them are independently deployable applications.

Account service is used to retrieve/create/update user accounts. Account service sends message to RabbitMQ topic exchange when new account is created. 

Store service is used to retrieve/create/update store info and store openings hours.

Notification service sends email messages. Notification service listen on topic for incoming account and appointment messages and then process them by sending welcome/reminder emails.

Appointment service is used to retrieve/create/update clients appointments. Appointment service sends message to RabbitMQ topic exchange when new appointment is created.

Each microservice has it's own MongoDb database. We created small Angular+AngularMaterial UI for that POC.

![](https://media-exp2.licdn.com/dms/image/C4E12AQHMM3PdjFQnJg/article-inline_image-shrink_1500_2232/0?e=1584576000&v=beta&t=niDZzMZVIqjNnNnYEB9y-Jv4ZdBe7zExaf-WZ8RfxlE)

This solution has a number of benefits:

-   Each microservice is relatively small, easier for a developer to understand
-   Easier to scale development. Each team can develop and deploy their services independently 
-   Improved fault isolation. If one service fails then only that service will be affected, other services will continue to handle requests
-   Each service can be developed and deployed independently
-   It provides loose coupling between collaborating processes which running independently in different environments with tight cohesion

Spring Cloud Stream is a framework for building event-driven microservice applications.

Let me start with a few words on the theoretical aspects of Spring Cloud stream. It provides three predefined interfaces out of the box:

-   Source -- can be used for an application which has a single outbound channel
-   Sink -- can be used for an application which has a single inbound channel
-   Processor -- can be used for an application which has both an inbound channel and an outbound channel

You can add the @EnableBinding annotation to your application to get immediate connectivity to a message broker(RabbitMQ), and you can add @StreamListener to a method to cause it to receive events for stream processing. The following is a simple Source and Sink applications which send/receives external messages.

//Source
@EnableBinding(Source.class)
public class AppointmentServiceImpl implements AppointmentService {

	@Autowired
	@Output(Source.OUTPUT)
	MessageChannel output;

	@Override
	public Appointment create(Appointment appointment) {

		Appointment existing = repository.findById(appointment.getId());
		Assert.isNull(existing, "appointment already exists: " + appointment.getId());

		repository.save(appointment);

		log.info("New appointment has been created: " + appointment.getId());

		this.output.send(MessageBuilder.withPayload(appointment.getId()).build());

		return appointment;
	}
}
//Sink
@EnableBinding(Sink.class)
public class RecipientServiceImpl implements RecipientService {

	@Override
	@HystrixCommand(fallbackMethod = "defaultReminderEmail")
	@StreamListener(Sink.INPUT)
	public void sendReminderEmail(String appointmentId){
		// Reminder Implementation
		log.info("Receive notification request reminder for appintment " + appointmentId);
	}
}

The @EnableBinding annotation takes one or more interfaces as parameters, interface declares input and/or output channels. Spring Cloud Stream provides the interfaces Source, Sink, and Processor or you can also define your own interfaces.

![](https://media-exp2.licdn.com/dms/image/C4E12AQF3lQOe7aOR4A/article-inline_image-shrink_1000_1488/0?e=1584576000&v=beta&t=dPWNinxzp2LZs11MyURbKPYkSMCcj86rspnEvIcd6JQ)

Spring Cloud Stream provides Binder implementations for Kafka and RabbitMQ. 

In addition you can configure how many brokers have to confirm receipt of your messages, before the request is returned to you with the parameter *acks*: by setting this to all you tell the broker to wait until all replicas have acknowledged your message before returning an answer to you.

We are using infrastructure services which help us to tie togethers our business services into event-driven microservices platform:

-   Gateway

Single entry point into the system, used to handle requests by routing them to the appropriate backend service. Netflix Zuul is the front door for all requests from devices and web sites to the backend.

-   Config-Server

Spring Cloud Config is horizontally scalable centralized configuration service. Spring Cloud Config provides server and client-side support for externalized configuration in a distributed system. With the Config Server you have a central place to manage external properties for applications across all environments. Spring Cloud Config Server is HTTP, resource-based API for external configuration, can encrypt and decrypt property values and easily embed them in a Spring Boot application using @EnableConfigServer. 

-   Eureka-Server

Service discovery allows automatic detection of network locations for service instances and dynamic address assignment for auto-scaling, failures and upgrades. *Client-side service discovery* allows services to find and communicate with each other without hard coding hostname and port. With Netflix Eureka each client can simultaneously act as a server, to replicate its status to a connected peer. In other words, a client retrieves a list of all connected peers of a service registry and makes all further requests to any other services through a load-balancing algorithm. Eureka provides a simple interface, where you can track running services and number of available instances: [http://localhost:8761](http://localhost:8761/)

![](https://media-exp2.licdn.com/dms/image/C4E12AQGlz_j8xx1TsA/article-inline_image-shrink_1500_2232/0?e=1584576000&v=beta&t=J1sac3KCRefnpo-zrdayLvX-5LuMkhki0jyczMEVuN4)

-   EFK

Elasticsearch+Fluentd+Kibana - Centralized logging, Elasticsearch, Fluentd and Kibana stack lets you stream logs from each service, search and analyze your logs, utilization and network activity.

This article explains how to collect(stream) Docker logs to EFK ([Elasticsearch + Fluentd + Kibana](https://www.linkedin.com/pulse/streaming-docker-logs-via-fluentd-elasticsearch-kibana-sergiy-bruksha/)) stack.

By combining Elasticsearch + Fluentd + Kibana we get a scalable, flexible, easy to use log collection and analytics.

-   Auth-Server

Auth Server is used for user authorization and secure machine-to-machine communication inside a perimeter. The OAuth 2.0 authorization framework enables a third-party application to obtain limited access to an HTTP service, either on behalf of a resource owner by orchestrating an approval interaction between the resource owner and the HTTP service, or by allowing the third-party application to obtain access on its own behalf. Spring Cloud Security provides convenient annotations and auto configuration to make this really easy to implement from both server and client side. 

-   Ribbon

It is a client side load balancer which gives you a lot of control over the behaviour of HTTP and TCP clients. Load Balancing with Ribbon automatically integrates two Netflix utilities: *Eureka *Service Discovery and *Ribbon *Client Side Load Balancer. *Eureka *return the URL of all available instances. *Ribbon *determine the best available service to use.

-   Hystrix

It implements Circuit Breaker pattern, which gives a control over latency and failure from dependencies accessed over the network, we're able to consume services with included fallback using 'static' or rather 'default' data and we're able to monitor the usage of this data.

-   Monitor dashboard

Each microservice with Hystrix on board pushes metrics to Turbine via Spring Cloud Bus with RabbitMQ broker.

![](https://media-exp2.licdn.com/dms/image/C4E12AQGz5raGwWjtXw/article-inline_image-shrink_1500_2232/0?e=1584576000&v=beta&t=4fG0C_wEB5EnrntecIbWJtT238tEb7D4-zc-uXlAwxo)

-   RabbitMQ

It is the most widely deployed open source message broker. Message queueing allows web servers to respond to requests quickly instead of being forced to perform resource-heavy procedures on the spot.

![](https://media-exp2.licdn.com/dms/image/C4E12AQHXxR84x5E87w/article-inline_image-shrink_1500_2232/0?e=1584576000&v=beta&t=OCRnS1KcAF6AggcDOJ9Kt1QcjIw_B3VD838XaETJpcg)

Important endpoints

1.  [http://localhost](http://localhost/) - Gateway and Angular web application host
2.  [http://localhost:8761](http://localhost:8761/) - Eureka Dashboard
3.  <http://localhost:9000/hystrix> - Hystrix Dashboard
4.  [http://localhost:8989](http://localhost:8989/) - Turbine stream is a source for the Hystrix Dashboard
5.  [http://localhost:15672](http://localhost:15672/) - RabbitMQ management 
6.  [http://localhost:8888](http://localhost:8888/) - Config Server

Microservices Challenges

-   Difficult to achieve strong consistency across services
-   ACID transactions do not span multiple processes.
-   Distributed System so hard to debug and trace the issues
-   Greater need for end to end testing

You are going to start 10 Spring Boot applications, 5 MongoDB instances, RabbitMQ, Elasticsearch, Fluentd and Kibana. Make sure you have 8GB RAM available on your machine. Start your Docker containers using start-all.sh script in docker folder or with docker-compose. See you containers running on Kinematic.
