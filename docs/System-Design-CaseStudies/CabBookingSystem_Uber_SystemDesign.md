# Cab Booking System Design

## Functional Requirements
- See Cabs
- ETA & approx price
- Book a Cab
- Location Tracking

## Non-Functional Requirements
- Global
- Low Latency
- High Availability
- High Consistency
- Scale
  - 100 Million Users on a monthly basis
  - 14 Million Writes per day.

## High Level Design

![Uber+System+Design](https://user-images.githubusercontent.com/34048837/143076506-289bfcc5-602e-455f-8d3a-9be8cd1651a0.png)

City will be divided into segments and the CAB is pin-pointed to a particular segment depending on his last known location.
All the users will be connected to system through a user app which talks to a service called User Service. The user service sits on top of a 
MySql cluster db which stores all the information like user profile, trips information and so on. Also uses Redis for caching.

Requesting Cab 🚕 is possible through Cab Request service. Basically it makes a WebSocket Connection with the users app also places a request with 
Cab Finder.

The Driver communicates through the Driver App. They have something similar to a user service called driver service and it has its own MySql Db Cluster
for saving driver profile related information & trips information.(By calling the trip service). Driver service will expose an API call to the Drivers App and they will be able to look
at the payment information, this is provided by the driver service after it internally makes a call to the payment service.

The Driver App also talks to something called as Location Service through a series of servers again maintaining a web socket connection with these servers.
As the driver moves through the city, there location information is being sent to the Location service which queries the Map Segment (Map Service) and find out
which Segment does the driver belong to.

So, when the customer places a Cab Request Service, customer segment is calculated, driver segment is calculated and they are mixed and matched and that information
is processed by the Cab Finder and feed that information is passed on to the user app.

## Customer and Driving Coming together.

The Drivers App uses a WebSocket connection handler mechanism, this is suitable because, we need to know the drivers location every few secs. And it would
be a costly operation if every few secs connections needs to be opened. And at the same time if some pickup/customer information needs to be passed, these
pool of connections would help.

If a trip needs to be assigned to a particular driver depending on the Segment proximity with the user, he needs to be informed and Web Socket handler Manager
helps to pin point where the user is, and drop him the notification. WebSocket Manager clearly knows which Web Socket Handler is connected to which Driver app/driver.
If a driver comes online or goes offline the same information is immediately sent across to Web Socket Manager through the Web Socket Handler.

Now the location information is also handled through the WebSocket Handler service and it posts it to Location Service.
The location updates are very frequent and high volume and velocity of these updates needs to be stored in the Cassandra database. Cassandra scales well.
(Last Known Location, Route taken by the driver to fulfill the trip). 

Location service talks to the Map Service, it maintains segment, it gives ETA, it gives routes to be followed by A to B. Map Service is exhaustively covered in Google Maps
System Design.

Trip Service is the source of trip information. It sits on the top of a MySql and a cassandra database. MySql stores data about the trips which are about to
happen in the near future or the In-progress trips. Once the trips are completed they are pushed to the Cassandra. The reason why we are storing
trip information in the MySql db is it would have lot of imporatant information like Users data, drivers data and most importantly, last location time, Potential
distances, start times and end times. Payment information. (Lot of tables).
Every incoming trip and the information that comes with it needs to be stored in lots of important tables and hence is transactional in nature and must be good
with ACID.

Once the trip is completed we can move it to cassandra. Movement from MySql to Cassandra is taken care by Trip Archiver.

Most of the trip and location information is passed on to the Kafka and analytics will run on top of it which specifically looks at 
solving the problems of driver rechability and customer proximity.

Cab Finder service is provided with dest lat/log values from the users app. It contacts the location service and tries to find out with map service
which segment does this user belong to ? And find out corresponding dirvers in that segment.

The Cab Finder service once finds the drivers in the nearest segment may leverage Driver prioritization engine depending on the customer that has 
requested for the cab service. (Premium customer may get the best possible driver in the nearest segment).

The Cab Finder then contacts the WebSocket Manager and Web socket handlers and notifies the driver.

Once it is confirmed the trip service is updated with driver/customer details.

## Using Kafka

The Kafka cluster is getting a lot of information
- Location Updates events
- No Driver Events
- Trip Update Events

Once the trip is complete we need to initiate the payment to the driver that could be aggregated over a few hours/few days or something of that sort.
But we still need to store information about a potential payment.
The Payment Service will consume the trip completion kafka events and initiates the payment to the driver and those details are stored in MySql db cluster.

## Analytics
A Spark streaming cluster services run on kafka events. It can help detect which segments have scarcity of drivers and generates the dearth of drivers
and produces a heat map so that drivers could move into those locations.

The same data is pushed into Hadoop Cluster where a lot of Spark ML jobs. There are multiple things that can happen.
- User profiling (what kind of customer is he - regular/one-off)
- Driver profiling (regular/premium drivers)
- Driver priority (Rating/Customer Feedback/ETA's used in calculation of score)
- Fraud Engine (High correlation between a customer/driver to identify if certain incentive programs are being exploited)
- Maps Service (Same data could be inputted for Maps)





