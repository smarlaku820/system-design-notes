# WhatsApp (Chat Application)

## Functional Requirements

- One-To-One Chat
- Group Chat
- Messages can be text/images/videos
- Read receipt
- Last seen


## Non-Functional Requirements
- Very low latency
- High Availability
- No Lag
- Scale
  - 2 Billion Users
  - 1.6 Unique Users
  - 65 Billion Messages being sent across


## High Level Design Diagram

![Whatsapp+System+design](https://user-images.githubusercontent.com/34048837/142822962-bb8d6917-b737-409c-b830-6d5fa4d8c75d.png)

### Web Socket Handler

A Web Socket Handler is a server which keeps open connections for all the active users. Each machine has 65k ports, so it can handle 65k connections.
If we reserve 1000 connections for internal use and few ports for running other system related services, still the system can use upto 60k connections.
The WebSocket Handler servers are distributed across the globe so that the connections can be formed according to user proximity.

### Web Socket Manager
The WebSocket Handler talks to WebSocket Manager. It keeps track of which users/devices and their open/Active connections to WebSocket Handlers.
The WebSocket manager takes the helps of a redis db.
So, as soon as the device/user opens the app, he acquires connection to the WebSocket handler and that information is stored in the redis db with the
help of Websocket handler through websocket manager.

### Message Service 
WebSocket Handler also talks to the Message Service. Message Service is the repository of all the messages in the system. It can get messages
by MessageId/UserId and so on. (Which uses cassandra). We choose cassandra because of the query pattern and the volume of the messages.

As soon as the message is delivered to the user, the message is deleted from the system. (unlike FB). 

So, the flow is like this. Let us suppose there is a user1/device1 who wants to send a message m1 to device2/user2. This is what happens.
User1/Device1 is connected to WebSocket Handler1, he generates a message and this message is processed by the message service, (stores in the cassandra) db
and generates the message id, m1. WebSocket Handler1 then with the help of WebSocket Manager1 finds out where is the user2 connected. If it finds that it is 
connected to WebSocket Handler2 it will directly send the message to WebSocket Handler2 and asks it to deliver message to user2/device2.

In the same way the WebSocket Handler2 will inform through the WebSocket Manager to device1/user1 through WebSocketHandler1 that the message has been delivered
and/or seen by the user2/device2.

Like wise lets talk about a scenario where how offline users/devices retrieve there messages. As soon as they come online, they will query
the message service if there are any messages which are not read/seen by the user3. And they will get those messages, as soon as they get those messages
through the WebSocket Manager the corresponding users who sent the messages are informed that the user3 has seen those messages and they will delete such
messages from the system. 

### Group Messages from the Context of User

The WebSocket Handlers are light weight they are not usually concerned with group messages. And this communication is handled seperately.
So, the Group messages are handed down to Message Service saying that user1 wants to send a message m3 to group g1. The message service will presist
the message in the cassandra db. And along with that it will put that message into a Kafka Topic.

There is a Group Message Handler which is the consumer of kafka messages. This Consumer after consuming messages from the Kafka will interact with a service
called Group Service. The Group Service maintains/queries a MySql User Group DB Cluster which contains all the details about the group, like when is the group
created, who are the admins, what are the permissions, who is part of the group and so on.

The Group Service provides the group message handler with the user information of a particular group. The Group Message Handler will then remove the user 
from where the message has originated and makes a note of all the users to whom the message has to be delivered. It then finds out through the WebSocket Manager
where this list of users are, ie., to which WebSocket Handlers they are connected to.

And then Group Service will initiate the comms with the WebSocket Handler to send corresponding messages to the connected devices.


### Images/Videos & AnyOther content from the Context of User

The Multi-Media is encrypted/compressed at the source, at the user machine. And then the user app will talk to something known as an Asset Service.

Let us say a user u3, whats to send an image i1 to user u2. It will contact the Asset Service through a LB. 
The Asset service will presist the media on S3/CDN and returns the image id i1 which contains the URL or metadata about the image object
that was sent.
The Asset service will also inform the message service that the image i1 is stored and u3 wants to send i1 to u2. The Message handler
service persists that information to the cassandra db.

And now the communication happens as is through the WebSocket Handlers and WebSocket Manager Services.

Optimization technique-
Hashing happens to encourage re-usability of same multi-media content. this often happens during sporting events/festive events.
Same image is being circultated over and over again across different groups/users.

So, the user/device will ask the Asset Service if the piece of multi-media content is already there, if it is not there then only it will ask it 
to Upload it to the S3/CDN and it will percolate to Message Service and so on.

### User Service/Group Service/Analytics Service/Last Seen Service From the Context of the User App

User service manages user profile information with the help of MySql RDBMS cluster.
The information can be cached in redis cluster to fecilitate reads for the Group Service which would often look for active user information
in conjuction with the group information from its onw MySql RDMBS cluster.

Each action the user is doing on the App gives us a lot of information. And the user behaviour can be extracted by a spark streaming service
that is sitting on top of this app. And this can be used to clasify users based on the behaviours.

Last Seen service sits on the top of a cassandra cluster and talks to kafka. It consumes the user availability and stores that information into 
the cassandra cluster. This is a very high through put system so Db like Cassandra/HBase would be the right choice.
It manily tracks the User App related activity, ie., when did the user open the app/close the app and so on.


## Scale

Keep track of CPU/Memory of your services. Disk Utilization in the Cassandra/MySql and other databases. Thruput and Latency on your web services.
The Disk Utilization in Kafka and more importantly the Lag in kafka cluster. That is, if you see more lag in the delivery of the messages, then it is a
sign that you must consume those messages.

Auto-Scaling and Automatically doing of the things must be done in near REAL time otherwise it would lead to very bad user experience which will result
in users dropping out of the platform.


