# Netflix/Youtube System Design

Netflix needs no introductory trailer.
Lets jump into the design



### Functional and Non-functional Requirements

Functional Requirements

1. User should be available to view and play videos. The videos would have various quality and depending on user's device the video quality could be different
2. The user should be able to search video by name/actors etc.
3. The system should have a recommendation system

Non-Functional Requirements
1. Low latency
2. High availability
3. Highly consistent
4. Highly scalable

### Lets talk CDN

CDN stands for Content Delivery Network. Since we are streaming potentially huge files, the bandwidth would be used optimally if the content resides close to the user.
So for User A in Bangalore, the system should hit the CDN server(s) located in India, while for User B in Munich, the requests should be routed to CDN Server(s) in Germany.

Also, User C when viewing a video on mobile device should be served a lower resolution content compared to USer D which is watching on a 55' Full HD TV.

So when very rich quality media is acquired from distribution houses must be Transcoded against various formats/ quality settings and must be replicated to these CDN servers.
The video could be further chunked, so as, to steam smaller parts??

### User management

We can use multi-region ELBs behind DNS and geolocation based Route 53 to direct user request to API servers, which can query NoSQL databases to list the various video titles available
These NoSQL databases would be replicated across regions, and could also have some content specific to the region, eg German titles not available in India etc.

Once a title is selected, we identify the best suitable video format and redirect the user to nearest CDN location to stream the content.

While the content is being played on the user's device, the app will periodically send the current video's time to the server in order to provide the ability to Continue Watching


### Health Management

To check the health of our CDN servers, we might have a periodic polling, to ensure the CDNs are healthy
Also, if we receive some errors on the User Device, regarding video playback, the same could be sent to a kafka topic for logging and management

### Recommendation System

Video viewing patterns will be sent to another kafka topic, and Spark could be used for building ML recommendation systems.
Similarly, Most watched titles could be created using viewing patterns.

