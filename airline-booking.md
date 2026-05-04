# Airline Booking System Design

Considering the high level requirements, I can come up with three main micro-services, other than user-management, payment processor, analytics and notification
1. Flight Search Service: Allows users to search for flights by providing the origin, destination, and travel dates.
2. Flight Comparison Service: Allows users to compare different flights based on factors such as price, duration, and airline.
3. Flight Booking Service: Allows users to reserve a flight and provide their personal and payment details.


# User Management
Simply allows user to create and manage account, retrieve flight bookings etc  
Lets keep it out of scope for this analysis

# Payment Processor
Interface with external systems for payment management, refunds etc    
Lets keep it out of scope for this analysis

# Analytics and Notification
While being a very important part of the final solution, for the sake of this discussion lets keep these out of scope, apart from mentioning the usual candidates like    
* Amazon Kinesis: Stream user interactions for real-time analytics.
* Amazon Redshift: Perform complex queries on booking and sales data for business intelligence.
* AWS Glue: ETL service for preparing data for analytics.


### Ideally, when the user makes request, we would use websockets and start returning results to the user from each service.


# Flight Search Service
From the user-entered source, destination, and time range fields, we invoke the flight search service.  
For the flight search service, we need a fast lookup for the Source to Destination. **The complexity lies in the fact, that there might be cheaper flights available between the two destinations with one or more hops in between.**

However to compute these hops for each request is **a no-go**, simply because of the load it will put on the system for each user requirement.

Since flights donot change that frequently, we should pre-calculate and store these paths in a database. In case we encounter a new source-destination combo, we can fire up a new Computation service and generate the paths by simple traversing a tree.

I am thinking a Graph database, like NeptuneDB or neo4j can store the pre-computed source destinations combo. The idea being the look-ups would be quick


# Flight Comparison Service
Once we have the various paths available from our Flight Search service, we should query our database which has Source-Destination as key and retrieve the Duration, CurrentPrice, Availability and other fields.

This will return to us for each Graph Path, a list of various flight options with each having its own duration and current price.

While SQL is the industry-standard for this kind of query, we could also use a key-value database like dynamodb. The reason I mention dynamoDB is because, with SQL databases also you need to do a Key lookup, there is no scan option that we can use without making our query too complex.


# Flight Booking Service

Once the user has selected a flight path to book, we route the request to the FlightBooking service, which takes care of interacting with other services like Payment Processor, user management etc.

However a contention could be the flight needs to be reserved for the user, for say 15 minutes. For this we could use a locking mechanism on a seperate table. 

A simple mechanism for this could be via a DynamoDB table as shown below


### Schema Design:
* **Partition Key**: FlightID_DATETIME
* **Sort Key**: Seat Number (e.g., 12A).
* **Attributes**:
    * LockExpiresAt (Timestamp): Time when the lock expires.
    * UserID: ID of the user who locked the seat.
    * Status: Indicates if the seat is "locked," "reserved," or "available."
  
### Locking Logic
1. **Check Availability**:
* Query DynamoDB to see if the seat is locked or reserved.
* If Status is "available" or if LockExpiresAt < current time, proceed to lock it.

2. **Place Lock**:
* Update the seat's status to "locked" with the LockExpiresAt set to the current time + lock duration (e.g., 10 minutes).
* Use DynamoDB's Conditional Writes to ensure atomicity. For example:
```sql
ConditionExpression: "attribute_not_exists(Status) OR LockExpiresAt < :current_time"
```
3. **Auto-Expire Lock**:
* Design the application to respect the LockExpiresAt. If the user doesn’t complete the booking within the lock duration, the seat automatically becomes available again.

### Booking Confirmation
1. If the user completes the booking within the lock duration:
    * Update the seat status to "reserved" and associate it with the booking record in the database.
    * Example attributes in DynamoDB:
        * Status: "reserved."
        * BookingID: Unique identifier for the booking.
2. If the user fails to book:
    * Leave the system to auto-expire the lock based on LockExpiresAt.

## Futher thoughts

1. The "Data Freshness" vs. "Pre-computation" Dilemma

    We discussed storing Pre-calculating paths in a Graph DB.

    * **The Problem:** Airline prices and availability change every second. If you pre-calculate a "cheap" path, by the time the user clicks "Book," the seat might be gone or the price doubled.
    * **The Fix:** Use a **Multi-Level Cache Strategy**.
        * Level 1 (Graph DB): Static routes (e.g., London to NYC exists).
        * Level 2 (In-memory Cache/Redis): Real-time availability and pricing for the next 24–48 hours, refreshed via a stream from the GDS (amadeus/sabre).
        * Level 3 (Direct GDS Call): Only at the final step of the search or during the booking flow to ensure the price is "locked" with the airline.

2. Handling Distributed Transactions (The Saga Pattern)  

    In a microservices architecture, a failure in the Payment service after you’ve reserved the seat in DynamoDB creates a "zombie" reservation.

    **We need a Saga Orchestrator**.

    If the payment fails, the Saga must trigger a "compensating transaction" to release the DynamoDB lock and notify the user. Without this, our seat inventory will become inconsistent.

3. Search Optimization: Read-Heavy Traffic

    Flight searches are typically 1000x more frequent than actual bookings.

    - **CQRS (Command Query Responsibility Segregation):** We strictly separate the "Search" (Read) side from the "Booking" (Write) side.
    - **Aggregator Service:** Since we are pulling data from multiple airlines, we need an Aggregator Pattern to merge, de-duplicate, and sort results before they hit the Comparison Service.

4. The "Thundering Herd" Problem
    
    When a deal goes viral (e.g., $200 flights to Hawaii), thousands of users might hit the same FlightID_DATETIME partition in DynamoDB simultaneously.

    - **Potential Issue:** Even with Conditional Writes, you might hit partition throughput limits.
    - **Optimization:** Implement Client-side Debouncing or a Request Queue (like SQS) to buffer the "Lock" requests if the system detects a massive spike for a specific flight.