# URL Shortner System Design

Functional Requirements
- Given a Long URL return short URL
- Redirect to Long URL when accessing short URL

NFR
- Very low latency
- Very highly available

Not catering to the NFR's often results in a poor user experience.

Questions to ask ?

- Whats the incoming traffic ? (How many URL's for shortening)
- Until what duration should this short url be saved ?

X rps (requests per second)
X * 60 (min) * 60 (hr) * 24 (day) * 365 (year) - rpy
X * 60 (min) * 60 (hr) * 24 (day) * 365 (year)* 10 = Y - rp10y

- You must build the system which can handle X * 60 (min) * 60 (hr) * 24 (day) * 365 (year)* 10  unique requests.
- CharacterSet that is allowed for shortening the URL.

Example, you could assume 0-9,a-z,A-Z Upto 62 characters in your short url.

If the length of the short URL is just 1 character you could at max serve 62 Long URL requests. Each character can represent a unique Long URL request.

Like wise if the short URL length is 2 then you could represent a max of 62^2

if the short URL length is n then you could represent a max of 62^n unique Long URL's.

Now, 62^n > Y (to satisfy the requirements)

n > log(y) base 62

== Architectural Design

UI -> LB -> Multiple instances of Short URL service (generates a short url) -> Cassandra

How do you ensure that multiple services doesn't collide or give the same short url, 

You can depend on a Redis cluster to give you that number. Causes Single Point of Failure. And when you implement multiple redis instances tied 
to url shortering services you have to manage redis cluster again, which can increase the latency and cause a bad User experience.

So, its better to use a token service (MySql) where the URL short service requests for ranges during their boot time.


<img width="704" alt="Screenshot 2021-11-20 at 6 51 01 AM" src="https://user-images.githubusercontent.com/34048837/142709295-47af8d3a-fe09-4969-9e7f-641c10508ce3.png">

<img width="644" alt="Screenshot 2021-11-20 at 6 52 08 AM" src="https://user-images.githubusercontent.com/34048837/142709296-85e8c165-b0c1-4964-b480-5373e5fda7fa.png">

And also you could use Apache Kafka for analytics, using Asnyc calls not for every request but by taking the help of a DS locally and flushing
the changes to Kafka frequently.

Apache Kafka can dump information to Hadoop and we can have Hive Queries written on top of it.
Or 
You could have a Spark Streaming service on Apache Kafka run the analytics and dump that information else where.




