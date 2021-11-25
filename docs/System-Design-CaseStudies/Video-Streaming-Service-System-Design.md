# Netflix, HotStar, Amazon Prime, Youtube

## Functional Requirements

- Upload Videos (users like you,me or production houses)
- User's HomePage + Search
- Play Videos
- Support all devices
  - This is a huge requirement as not all devices support and/play all video formats.
  - So, this boils down to supporting all the video formats
  - Formats usually co-exist with dimensions and screen sizes, aspect ratios
  - So, this is a combination of (Types of Devices * Video Formats Supported * Dimensions of the Screen * BW) with High/Medium/Low BW regions.
  - To reduce the file size or to cater to BW requirements, you need to reduce the video size by adjusting the Frames Per Sec.

## Non-Functional Requirements
- No Video Buffering (Video should be available to the user with Low Latency)
  - Low Latency
  - High Availability
- Increase users session time 
  - Recommendations

One important thing to note is a device never downloads an entire video. Video is split into chunks and buffered.
The client is intelligent enough to request for a specific format of the video depending on the quality requirements and other preferences
for playing video. Depending on the BW requirements the client may adjust the quality of the video accordingly. Once it has requested
enough chunks and finds that it can switch back to high quality content it may do so. This is called __Adaptive Bit Rate Streaming__.
There are libraries which can implement in __ABRS__. But most of the devices have this feature baked in.


## Understanding the Users of the Platform
- Clients (Device Type - Tablet/PC/TV)
- Users (Human viewing the content)
- Production House (Humans)

## High Level Architectural Design

![Netflix+System+Design](https://user-images.githubusercontent.com/34048837/143183624-de47c4a3-9e35-4a7d-bc8a-4af71d32a280.png)

Let us understand the architecture step by step. 

### Content Uploading and Asset Onboarding Service

The Content creator or the production house upload a video usually by providing a secure FTP link and the Asset Onboarding service
will take care of onboarding it into S3. The MetaData for the videos is stored in Cassandra Db Cluster. (Who uploaded it, thumbnails, tags and other
important metadata).

Asset Onboarding Service will post an event into kafka and informs that a particular video content has been uploaded into S3 and metadata into Cassandra

### Content Processor/ Asset Service

As discussed in the functional requirements, playing a video on a end users machine is not a straight forward task, there are so many aspects
regarding device type, bw, dimensions, formats etc.,
The Content processor has a lot of tasks, just to prepare for the above.
The very first thing it does is to Chunk the video file into parts.

All the chuncks will go through the Content Filter which will look out for Piracy, Nudity and Legal. The Content processor in each and every stage sends events to the Kafka cluster.

Then the content will be tagged.(classifier, thumbnails).

After content tagging it will go through Transcoder.  (An activity of converting a single file into multiple formats or files)

Each of the file formats will then be converted into variying qualities. (Quality Converter)

And this will be uploaded into the CDN. (individual components).

While the Content Processor at every stage is doing its job on a specific video it keeps sending this data to the Kafka Cluster. There is a Spark
Streaming service which keeps aggregating files at each stage and may be used to create tags/thumbnails and so on.

And at the same time the entire video content processing related events are consumed by the Asset Service and continously persisting such data
into the Cassandra Db Cluster. 

After it is successfully uploaded into CDN, using notification service a notification will be sent to the user who uploaded the video.

## Powering the User who tries to access the video streaming app

User will login and user information will be powered by the User Service and it uses Redis as a cache. (And backend as MySql db)
User service will also become a source for content filtering/age restrictions and so on and usually used/invoked by the search service.

The user login activity is also logged into Kafka for important analytical functions
- User Finger printing, subscription based model, is he logging in from multiple devices from various geographies ?

The Home Page service will give the data as soon as the user logged in.
The user behaviour is again streamed into kafka (with Analytics Service) and the video streaming service needs to understand if the best search results are
being utilized or not ?

Then there is User actually playing the video, this will be helped by a Host Identifying service with the help of Asset Service & Cassandra Cluster.
It will come to know across what CDN's are the file chunks available to play and in what order (with the help of cassandra) and renders the same to 
the user. (Returns a Main CDN and CDN Optimial for Local Views if a lot of devices are requesting for a specific content to cater to Low Latency)

How good a video is and how is it streamed while playing ? And how long the people are watching ? And these are done by Streams Stats Logged Service
which is published to Kafka. And analytics is run on top of it to rate the video.


## Kafka And the Analytics

Asset Service uploading video processing events into Kafka. Immediately Kafka will make it available for searching. Search Consumer Service.
Which will then be used for powering Elastic Search.

And when the user searches from their home screen, based on different attributes, the video title, or the actors in the video or by the tags applied on the video
The ES will make all of these available and the Search Service knows this.

If the Search service wants to apply any filters, it will query the user sevice and apply those filters (like Age)

The Hadoop Cluster is used for aggregating let us say best thumbnails for a video, this will be a culmination of stream of events from Asset Service (content processor sending events) and the user Analytics Service.
ML can be run and we can say which kind of user liked what kind of thumbnail ?


User viewing a video for a particular length of time is also streamed into Kafka. And if the user searching is recorded into kafka and
all of this information will be used for improving the search recommendations and user content needs.

This work is done by Apache Spark Cluster (Recommendation Engine) and that information will be presisted in Cassandra Cluster and will be powered
or accessed by User Home Page Service.

Usually we classify the users based on the geners they watch and we improve the recommendations of the other users by the likelyhood of same user searching
behaviour.

Traffic Predictor come up with a list of videos the following day, you could copy the content from Main CDN to Local CDN. Predict what could be the possibility
of users seeing ? Traffic Predictor puts those events into the Kafka.

A CDN Writer will do its job depending on the recommendations of the Traffic Predictor.

The Local CDN's will get only specific chunks of data and other remaining chunks are pulled from other local CDN's the CDN's will work together.

Optimizations at ISP Layer

If there are number of users who are coming through the same ISP route, the ISP can choose to download the chunks at one central place
and cater to the user needs. They need to look at the trade off, (BW vs Maintaining the infra for persisting those frequently asked chunks.)

### Open Connect (OC)
The Video streaming service companies like Netflix will take off HW (Open Connect Appliance (OC)) from their CDN's and work with ISP's to provide them a caching service.





