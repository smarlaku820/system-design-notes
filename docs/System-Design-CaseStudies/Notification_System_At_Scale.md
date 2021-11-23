# Notification System (At Scale)

## Functional Requirements

- Send Notifications
- Pluggable (ex:- SMS/Email initially and Add App Notifs later, extensibility should be taken care off)
- SaaS (Rate Limiting)
- Prioritization (setting high priorities for different kinds of messages)

## Non-Functional Requirements
- High Availability
- Many clients (add clients and track requests)

## High Level Architecture

![Notification+Service+design](https://user-images.githubusercontent.com/34048837/142979117-c6639bac-71e3-4fa0-9778-12430e5874fe.png)

## Notification Clients & Notification Service
The starting point of the design is to focus on the clients who send notifications. There are two-three kind of requests that they send you.
- I want to send a particular content to this user via an email
- I have this user id, and want to send this particular content, you choose how you want to send. via an email or sms etc.,

Its good to have that the above usecases are handled by your solution.
They are handled by the Notification Service which handles notifications which are targeted to various other clients or groups or other companies. 


Notification service will put the request on Kafka and informs the client that the notification request will be taken care off. You could take it
as a transactional flow and make it more synchronous but better not to do that. Take the help of Kafka and make the notification requests Asnychronous.

Notification Service does basic validations, userid shouldn't be null, content should not be null. There are other important/custom validations
that will be done at Notification Validator Or Prioritizer. The main thing is setting the priority for the message. And this priority is decided
based on the attribute of the notification request like message type identifier.
- OTP's will have the highest priority
- Transactional Message (related to orders)
- Promotional Messages

Once the priority has been decided they will be put on different topics based on priority. So, the consumers at the other end also will consume the
high priority messages first then the medium and the low priority messages later. We don't want any lag on the high priority messages
but some delay on low priority messages can be ok.

## Rate Limiter/Request Counter
The next component is a Rate Limiter. What a Rate Limiter should decide is, if there is a client who is calling me, am i supposed to get called
these many number of times ? And i am supposed to send these notifications to a particular user these many number of times.

Or
Subscription to a client is like max 10 messages/sec.
It can look like a configuration piece that get applied, that you cannot send a notification to a user more than 3 times per day.

So, they are thresholds set based on configurations and after the thresholds are set the requests are dropped. Or there could be other thing altogether
where you will be charged based on the number of notification requests and there will be no rate limit applied.

The Rate Limiter could use a Redis Cluster. 

## Notification handlers and Vendors


The next component is called a Notification handler and user preferences. The user may have a choice depending on what kind of messages he wants
so, those settings will be applied. Or the user must have said, i would like to unsubscribe from all your promotional messages & it would 
take the help of a MySql preferences db which is built on top of user preferences.

And it will also search the external User Service to get the user to get more details about the user.
And we have another Kafka and there are different topics (email/sms/IVRS/In-App) and we need kafka messaging broker because there can be so many SMS's
at one go that it may break a service. You could add more nodes/pods so that the service could handle, but just because certain clients are having spikes
it is not worth it.

Put it in Kafka and the Handlers could handle at their own time.
Vendor Integrations are done by configuring the Handlers.

In-App Handlers are for sending notifications through App's via Firebase (Android) and Apple Push Notification Service (iOS).

IVRS (Interactive Voice Response System) calls are voice calls that are usually set up when you have placed high transactional Cash-On-Delivery via an e-commerce
website. IVRS is again a low throughput scenario. And the Handler Services usually tie up with Vendors to deliver those calls or notifications.

## Plugabbiilty
If you want to add a new messaging extension, you can add it very easily and the above architecture easily extends it. This is what you must/can do.
- Add a WhatsApp Handler
- Add a Notification Type as Whats App Messaging at the very beginning (Notification Service)
- And it flows through the Validator -> Kafka -> Rate Limiter -> Notifs preferences -> Kafka (WhatsApp Topic) -> WhatsApp Handler (WhatsApp Consumer)

## Notifications tracker
For keeping a track of all notification services. You need to keep a record of what all has been sent.
You could use a Cassandra Db Cluster for this regard. This is mostly Write-Only mechanism.

The components can be squeezed depending on the throughput. It depends on the use case and the requirements.

## Bulk Notifications

Like All the users who ordered TV in the last 24 hours, send them a notification.
Or for all the people who ordered pizza or milk 3 days back check if they want to order the same item over the weekend ?

You have an UI and a Bulk notification service which looks at the notification and the filter criteria and sends it out.
Filter Criteria (Find all the users satisfying a particular criteria) ? This data is obtained from User Transaction Data service. This is the important
service which abstracts a lot of Business Critical or transactional services which are outside the context of Notification System.

And the order information is dumped into Kafka topics and a search functionality can be added on top of those topics. 
And that is how your filter criteria is satisfied parsed and queried with the help of a query engine.

Parser is important because there could be various signatures of transaction data and it must be parsed into a specific format
that can be useful and will be used to fed the search engine. 
The query engine will have all the list of queries.
- Like find out all the list of users in a specific location who have orders in their basket but orders haven't been placed for more than 3 days.

And this query engine have a lot of other consumers, there can be fraud engine/rule engine and so on.
query engine would often return a list of users matching a filter criteria.
- Rule Engine (If a user has cancelled 10 consequtive orders suspect that user and mark that account deactivated)
-   Or it can be that after 5 days you have received the order, then can send a notification asking for feedback.
- Fraud Engine can read through the transaction data and if some filters apply it could take action based on that.

And then the Bulk notification service would get the list of users which are obtained from the filter criteria and send those messages to the
list of users.
