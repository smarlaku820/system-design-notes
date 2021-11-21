# Air BnB

## Functional Requirements

Hotel -
Onboarding
Update (room/pricing/images)
See the bookings
Insights


User -
Search for a property
Filters
Book
Check Bookings

Analytics -


## Non-Functional Requirements

- Low Latency
- High Availability
- High Consistency (Booking a hotel, see that hotel)
- Scale ( 500k Hotels | 10 Milliion Rooms | at least 1000 rooms per hotel  (Max.7500))

## Overall Design


![Airbnb+System+Design](https://user-images.githubusercontent.com/34048837/142763182-131e2b03-08b9-44a3-a6d8-fd9162ad4ccf.png)

## Functional Flow

### Hotel

Hotel data is relational so an RDBMS MySql Cluster database will be used.

All the Hotel images will be used dumped into a CDN and the Image URL's are stored in MySql db cluster.

Hotel Service is a Horizontal Cluster of nodes where it runs.

Each modification happening in the hotel service, we need to bubble the hotel information to the users And that is done via Kafka.

### Kafka Cluster and ES Cluster

The consumers of Kafka will populate the search store (Elastic Search Cluster which supports Fuzzy Search) and thus it caters to the search traffic.

Any Spike in the user traffic, the nodes in the Kafka Cluster/ES cluster can be increased horizontally.

### Search Service

Search Service powers the search along with tags & filter criteria.

### Booking Service & Payment Service

Booking service sits on a RDMBS Cluster.

Whenever a booking happens it will be stored in the MySql & talks to a payment service. Once the Payment is confirmed, the booking is confirmed.

__Note:-__ There is a seperate MySql cluster used for Bookings and Hotels, this has been intentional although we could use the same MySql Cluster.
This has been done so that the scale depending on the traffic can be handled without affecting each other.

Once bookings are done the same information is sent back to the Kafka and the search consumers will remove the such rooms/hotels booked from the
search criteria within that date range.

Usually the booking information is stored in the MySql cluster and only if they a specific booking has reached a terminal state like CANCELLATION/
CONFIRMED it will be persisted in a Cassandra database through the Archival service. The Archival service will get its feed from Kafka as well
as the MySql cluster.

Cassandra is very good for filter queries on a specific key (partition). 

### Notification Service

Once the bookings are completed, we need to notify Hotels/Customers. (or when there are changes to the booking)

When the hotel bookings are cancelled a notification will be sent to customers.

### Booking Mgmt Service
Users and Hotels might want to see their bookings. A Read only view.

There are two data sources which are powering the booking mgmt service the MySql cluster where a Redis will be used in between to reduce
the load on the MySql cluster for all Active Bookings. 

And the Cassandra cluster will be used for all the bookings which are Confirmed/Cancelled.

### Hadoop Cluster

All the events from kafka are pushed onto Hadoop.

A Spark Streaming service will read from Kafka and loads data into Hadoop cluster. Then you can write all the Hive Queries to pull various
Analytical details.

## Low Level Details





