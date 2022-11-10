
In a previous article,
[we learnt all about Microservice Architecture](https://github.com/khalifa005/MicroserviceArchitectureDemo)
In ASP.NET Core (I recommend reading this before continuing), API Gateways, Ocelot Configuration and much more. In this article, let’s talk about another aspect – Microservice Communication using RabbitMQ with ASP.NET Core. Essentially, we will learn how to enable communication between Microservices using RabbitMQ and MassTransit. Let’s get started!

# What is RabbitMQ?

RabbitMQ is one of the most popular Message-Broker Service. It supports various messaging protocols. It basically gives your applications a common platform for sending and receiving messages. This ensures that your messages (data) is never lost and is successfully received by each intended consumer. RabbitMQ makes the entire process seemless.

In simple words, you will have a publisher that publishes messages to the message broker (RabbitMQ Server). Now the server stores the message in a queue. To this particular queue , multiple consumers can subscribe. Whenever there is a new message, each of the subscibers would receive it. An application can act as both producer / consumer based on how you configure it and what the requirement demands.

A message could consist of any kind of information like a simple string to a complex nested class. RabbitMQ stores these data within the server till a consumer connects and take the message off the queue for processing.

Additional, RabbitMQ provides a cool Dashboard for monitoring the messages and queues. We will be setting up this too later in this article!

# Advantages of RabbitMQ

There are quite a lot of advantages of using a queue based messaging solution rather that directly sending messages to the intendend consumer. Here are few of the advantages.

Better Scalability – Now, you will not have to depend on just one VM / processor / server to process your request. When it get’s to the point where you first server finds it tough to process the incoming queue data, you can simply add another server that can share the load and improve the overall response time. This is quite easy with the queue concept in RabbitMQ.
Clean User Experience – You users are less likely to see any errors, thanks to the microservice based message broker architecture.
Higher availability – Even if the main Microservice is down to a technical glitch on on-going update, the messages are never lost. It gets stored to the RabbitMQ server. Once the Service comes online, it consumes the pending messages and processes it.

Here is a simple demonstration of work-flow in a basic RabbitMQ setup. Here consumer #3 is offline for a specific time. This does not in any way affect the integrity of the system. Even if all the consumers are offline, the messages are still in RabbitMQ waiting for the consumers to come online and take the message off their particular queues.



![image](https://user-images.githubusercontent.com/29863643/201127543-f364b663-0b47-4a43-b22b-3384e8c5fd55.png)

# What we will build?

Let’s mimic a ticketing application where the user can book his/her ticket. We will have 2 Microservices with RabbitMQ connection for communication. The user buys a tickets via the front-end. Internally, this generates a POST request from the Ticket Microservice. The ticket details would be sent to a RabbitMQ Queue, which would later be consumed by the OrderProcessing Microservice. These details will finally be stored to a database and the user will be notified by email of the order state. We will keep it quite simple


So, why RabbitMQ in this scenario? Can’t we Directly POST to the OrderProcessing Microservice?
No. This defeats the purpose of having a Microservice architecture. Since this is a client facing, business oriented application, we should ensure that the Order always get stored in the memory and doesnt fail / get lost at any point of time. Hence we push the order details to the Queue. What happens if the OrderProcessing Microservice is offline? Absolutely nothing! The user would be notified something like “Tickets confirmed! Please wait for the confirmation mail”. When the OrderProcessing Microservice comes online, it takes the data from the RabbitMQ Server and processes it and notifies the user by email or SMS. That’s some great User Experience, right?


# Setting up the Environment

We will be working with ASP.NET Core 5 WebAPI using Visual Studio 2022 IDE. Make sure you have them up and running .

After that, we will need to setup the RabbitMQ server and dashboard.

# Installing ErLang
Erlang is a programming language with which the RabbitMQ server is built on. Since we are installing the RabbitMQ Server locally to our Machine (Windows 10), make sure that you install Erlang first.

Download the Installer from here – https://www.erlang.org/downloads . At the time of writing , the latest available version of Erlang is 23.0.1 (105 Mb).

Install it in your machine with Admin Rights.


# Installing RabbitMQ as a Service in Windows
We will be installing the RabbitMQ Server and a service within our Windows machine.

Download from here – https://www.rabbitmq.com/install-windows.html. I used the official installer. Make sure that you are an Admin


# Enabling RabbitMQ Management Plugin – Dashboard
Now that we have our RabbitMQ Service installed at the sysyem level, we need to activate the Management Dashboard which by default is disabled. To do this, open up Command Prompt with Admin Rights and enter the following command

```ruby
cd C:\Program Files\RabbitMQ Server\rabbitmq_server-3.8.7\sbin
rabbitmq-plugins enable rabbitmq_management
net stop RabbitMQ
net start RabbitMQ
```

Line #1 we make the instalation directory as our default working directory for cmd. Please note that your directory path may differ.
Line #2 – Here we enable the Management Plugin
Line #3-4 – We restart the RabbitMQ Service.

![image](https://user-images.githubusercontent.com/29863643/201138018-8dd33103-64f5-4e7a-8e58-8671e6e3a690.png)


That’s it. Now Navigate to http://localhost:15672/. Here is where you can find the management dashboard of RabbitMQ running.


![image](https://user-images.githubusercontent.com/29863643/201138349-e0674e7e-5ebc-49a5-99c9-0f09c1ebf0d3.png)
http://localhost:15672/ – This is the default port of RabbitMQ


The default credentials are 
guest
guest 
Use this to login to your dashboard.


![image](https://user-images.githubusercontent.com/29863643/201138492-dcb95eb6-dd73-4b17-9879-2f17024a97e1.png)

You can see that our RabbitMQ Server is up and running. We will go through the required tabs when we start setting up the entire application. However, you can manage users via the Admin Tab.


# Getting Started – RabbitMQ with ASP.NET Core

Now that our server is configured, let’s build the actual microservices that can interact with each other via RabbitMQ. Before proceeding, 
I highly recommend you to [go through Microservice Architecture](https://github.com/khalifa005/MicroserviceArchitectureDemo)  in ASP.NET Core with API Gateway to get a basic idea on how Microservice Architecture works. We also discussed about API gateways with Ocelot Configurations.

So, i created a new Blank solution ‘Microservices.Ticketing.WebApi’. Here we will be adding 2 Microservices

Ticketing Microservice.
Order Processing Microservice.

Since this is going to be a more RabbitMQ centric implementation, I will skip the Database Access and Email Notifications for the user. We will concentrate of sending and receiving the ticket order information only.


# Shared Model Library
Create a new .NET Core Library Project and Name is ‘Shared.Models’. Here we will define the shared models, which in our case is a simle Ticket Model.

# IMPORTANT – Note that it is necessary to have a shared Model class that is used by both the Publisher / Consumer. This is so because RabbitMQ treats messages based on it’s namespaces. if the received message and outgoing message are of different namespace (signatures), RabbitMQ would not recognize the Consumer.

