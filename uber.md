# Uber System Design

User/OLA/Lyft allow you to book cabs on the fly, it has both a User Side (to enter destination from current location), 
as well as, a Driver Side (to accept trip). Payments will be out of scope for this system design exercise though.


### Functional and Non-functional Requirements

Functional Requirements

1. User: Send booking request, See cabs in your vicinity
2. Driver: See booking request when idle (or nearly completing a trip), accept request
3. ETA and approximate price
4. Location tracking when trip is in progress
5. Payment from User to System and from System to Drivers (out of scope)

Non-Functional Requirements
1. Low latency
2. High availability
3. Highly consistent
4. Highly scalable

Note: : High availability and consistency at the same time seem to violate the cap theorem which states that a distributed system can either be highly available or consistent. But not all the components need to be consistent and available at the same time, some need to be consistent, and some need to be available.

To work with geolocations, we can again use the concept of GeoHash as described in the [Google Maps System Design](google-maps.md)  
The same concepts for route planning and ETA will be used here.

Lets have a look at how customers and drivers interact with the system.

* All Users will interact with the system via a user app which talks to the user service. User service ses a **SQL DB** at the backend and stores all user related information (name, ratings, billing info etc )
* All drivers also similarly interact with another *SQL DB* containing driver information via a driver service. 
* All the drivers' current location is also periodically send to the system (when on duty) and sent to a **In-Memory Cache (Redis)** where the **primary key is the Geohash (with TTL)**. This is achieved via WebSockets over a LocationService

When a user starts a booking process, 
- his location is sent to the CabRequest Service, which send the data to a **kaka topic**.   
- A CabFinder service is reading from this kafka topic (to make it scalable), it takes the booking request and tries to find the nearest available drivers based on Geohash.    
- **This query can first start in the current geohash and then expand to adjoining geohashes to find available drivers.**  
- The CabRequest service also starts a **WebSockets connection** with the user's app and periodically sends available driver locations to the app for the user to see and ETA is shown.

When the CabFinder service, find one or more available drivers, the request is sent to the drivers or them to accept.
Here we must use **DB transactions or Zookeeper election mechanism** to ensure that only one driver is able to accept the request.

Once the driver has accepted the request, we send the details to the Trip Service, which tracks the ETA and the start of trip.
For safety reasons, the user can also be sent an OTP which he has to provide to the driver to Start/Stop the trip.

The Trip Service keeps updating a **Kafka Cluster** with periodic location for analytical purposes.
Once the trip completed the information is sent to a Payment service to make the necessary monetary adjustments. 

### Analytics

As ou can see we are using Kafka to collect various events in the system. eg. time taken to find driver, the driver responses, the trip duration, billing etc.    
This can be used for 
1. analyzing hotspots or areas with scarcity of drivers.
2. Driver profiling, User profiling (further linked to coupons etc)
3. Fraud detection etc.
