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

### Hotel Service

Note:- This is not an exhaustive list of API calls

| API's | Description |
| - | - |
| POST /hotels | To create a hotel |
| GET /hotels/id | To get details of a hotel |
| PUT /hotels/id | To update details of the hotel |
| PUT /hotels/id/rooms/id | To update/create room details in a specific hotel |

<img width="316" alt="Screenshot 2021-11-21 at 7 14 04 PM" src="https://user-images.githubusercontent.com/34048837/142764389-2fdbaf45-f2d0-4ae1-ab35-964a2f954083.png">

Note:- This is not an exhaustive list of tables

The words highlighted in RED are either a primary key or foreign key.

`price_min` or `price_max` are the fluctuations in the price system depending on the price service.

<img width="482" alt="Screenshot 2021-11-21 at 7 16 28 PM" src="https://user-images.githubusercontent.com/34048837/142764392-01446bc0-95df-46b6-875e-db23e65444d0.png">

The information in the MySql hotel database is not powered by a redis caching service, this is because this is not subject to any High Thruput biz traffic.
Otherwise, the API calls must have been much faster. Trade-off analysis must be done, if you are adding infrastructure to get certain benefit.

### Booking Service

The Post API will perform specific steps. Such steps require ACID support and it requires MySql or an RDMBS database support.

<img width="548" alt="Screenshot 2021-11-21 at 7 26 54 PM" src="https://user-images.githubusercontent.com/34048837/142764771-57a0a1a5-7f4d-4871-a24b-60f5fceb481a.png">

<img width="367" alt="Screenshot 2021-11-21 at 7 26 25 PM" src="https://user-images.githubusercontent.com/34048837/142764767-0cccf506-4da9-4445-8dd6-0094186031f6.png">

<img width="581" alt="Screenshot 2021-11-21 at 7 34 09 PM" src="https://user-images.githubusercontent.com/34048837/142764977-51d19a64-b824-43d3-9607-3472c18b2a93.png">

Rooms cannot be RESERVED for an infinite amount of time. There will be time limit beyond which the rooms will be unblocked and transactions will be
reversed so that it is available for other users to book.

This can be achieved by utilizing the TTL feature in redis. The Booking UUID will be stored in redis which has that RESERVED status with a TTL.
As soon as that key is expired a callback will be issued & we can do whatever at that point in time.

In this scenario, the callback must revert the transaction in the booking table and increment the availble rooms to their original.

If Payment is Success the booking status will turn BOOKED from RESERVED & Invoice ID will be generated. (Kafka events could be sent)

If Payment is failure the booking status will turn CANCELLED from RESERVED. (Transaction must be revered the available rooms should be put back)

If Redis Key expired (TTL) callback is issued, booking will be CANCELLED, no invoice and the available rooms should be put back.

If Payment is success & Key expiry - two possible scenarios.

Key expiry call back comes after the booking is BOOKED, then no action is taken.

Key expires and the booking is CANCELLED and then comes the payment confirmation, in this case the payment must be reverted as booking cannot be done.

Or smartly check for the available rooms and see if you can re-play the booking as the payment is already received.

### Observations

Kafka scales much better than RabbitMq/AmazonQ
Cassandra is usually better choice for partition queries instead of HBase (lot of operational overhead). And more over
we have data shared by partition keys. There are two partition queries here, Get bookings by Hotel and Get bookings by user.
Monitor how CPU/memory behaving across all the micro-services. Disk Usage for Redis/Elastic Search is ?
Monitoring could be done through Grafana/Prometheus ? These must be done to achieve the NFR's for the setup.
Key metrics can affect Availability and Performance of your application in which case will hit the User Experience.
Resiliency/Disaster Recovery must be accounted for.
Divide your infrastructure based on the geography and have replicated DC's in multiple regions. This will help for high availability.



