## Design a site like cricinfo

### Functional Requirements
* Live match scores & ball-by-ball commentary
* Historical stats (players, teams, series, matches)
* Scorecards, rankings, news articles
* Search (players, matches, series)

### Non-Functional:
* High read traffic, especially during live matches (spikes to millions of concurrent users)
* Low latency for live updates (~1–2s delay acceptable)
* High availability (99.99% uptime during matches)
* Eventually consistent for stats: strong consistency less critical


###  Scale Estimation
| Metric | Estimate |
|---|---|
| DAU during major matches | 5–20 million|
| Peak concurrent users | 2–5 million |
| Read/Write ratio| ~1000:1 (very read-heavy)|
| Data size (historical) | ~10–50 TB |
| Live update frequency| Every ball (~30s intervals) |

### High-Level Architecture
```
Users → CDN → Load Balancer → API Gateway
                                    ↓
              ┌─────────────────────────────────────┐
              │  Live Score    │  Stats API  │  CMS  │
              │  Service       │  Service    │       │
              └────────────────────────────────────-─┘
                    ↓                  ↓
              Message Queue      Read Replicas
              (Kafka)            (PostgreSQL/
                ↓                 Cassandra)
              Cache Layer
              (Redis)
```

#### Live Score Service
* **Data ingestion:** Official scorer feeds (ESPNcricinfo uses official ICC/board feeds via proprietary APIs)
* **Push architecture:** Use WebSockets or Server-Sent Events (SSE) to push ball-by-ball updates to clients
* **Message broker:** Kafka to decouple ingestion from fan-out to millions of subscribers
* **Cache:** Redis pub/sub for live state; current match state always in memory
* **Polling fallback:** For clients that can't hold WebSocket connections

### Stats & Historical Data Service
* **Database:** PostgreSQL for relational stats (player averages, head-to-head, series records)
* **Read replicas:** Heavy read traffic routed to replicas; writes go to primary
* **Caching:** Redis/Memcached with aggressive TTLs — historical data rarely changes
* **Data model:** Denormalized for query speed (e.g., pre-computed career stats)

### Content/News Service
* Standard CMS-backed service (headless CMS like Contentful, or custom)
* Articles served from CDN edge nodes
* Pre-rendered static pages for evergreen content

### Search Service
* Elasticsearch for full-text search across players, matches, articles
* Autocomplete on player/team names
* Indexed on write via async event pipeline

## Database Design Highlights
```code
Key entities:
Match → Series → Tournament
Match → Innings → Over → Ball (ball-by-ball)
Ball → Batsman, Bowler, Fielder (FK → Player)
Player → Career Stats (denormalized aggregate table)
Team → Player (many-to-many via squad)
```

#### Partitioning strategy:
* Partition *ball_by_ball* table by *match_id* — queries are almost always within a single match
* Partition *historical* stats by *year* for archival queries

## Handling Traffic Spikes
During a World Cup final, traffic can 10–50x spike.

Strategies:
* **CDN-first:** Scorecard pages served as near-static HTML from edge (refreshed every 30s via ISR/stale-while-revalidate)
* **Read replicas auto-scaling:** Horizontal scaling of DB read layer
* **Circuit breakers:** Degrade gracefully — serve cached/stale scores if live feed is delayed
* **Rate limiting:** At API gateway level to prevent abuse
* **Queue-based fan-out:** Kafka consumers distribute live updates; don't hammer DB directly

## Live Update Flow (Ball-by-Ball)

```
Official Feed → Ingestion Service → Kafka Topic: "live-balls"
                                          ↓
                              Score Processor (consumer)
                                ↙           ↘
                      Update Redis         Persist to DB
                      (live state)         (async, batch)
                           ↓
                    WebSocket Server
                           ↓
                    Push to all subscribed clients
```

### CDN & Caching Strategy

| Content Type | Cache Duration| Strategy|
|---|---|---|
|Live scorecard| 5–15 seconds | CDN edge with short TTL|
| Historical scorecards| 1 year| Immutable, long TTL|
| Player profile stats| 1 hour| CDN + Redis|
| News articles| 10 minutes| CDN|
| Ball-by-ball| No cache| WebSocket push |

## Key Trade-offs
| Decision | Trade-off |
|---|---|
|WebSocket vs SSE vs polling| WebSockets = lower latency but harder to scale; SSE simpler; polling easiest but laggy|
|Cassandra vs PostgreSQL for ball data| Cassandra = better write throughput & horizontal scale;  Postgres = richer queries|
| Pre-computed stats vs on-demand| Pre-compute = fast reads, stale risk; on-demand = fresh but slow for complex queries|


When you're building a site like Cricinfo, where the primary goal is pushing data from the server to the client (one-way traffic), SSE is often the more elegant choice.
### SSE vs. WebSockets: The Breakdown
| Feature|	Server-Sent Events (SSE)|	WebSockets|
|---|---|---|
|Communication|	Mono-directional (Server to Client)|	Bi-directional (Full Duplex)|
|Protocol|	Standard HTTP|	Custom WS Protocol (requires upgrade)|
|Complexity|	Low (works with standard load balancers)|	High (requires specialized handling)|
|Reconnection|	Automatic (built into the browser)|	Manual|
|Scaling|Highly Scalable|	Resource Intensive|


**The SSE Advantage:** SSE is pure HTTP. To your infrastructure, it looks like a very long download. Standard LBs (like Nginx, AWS ALBs, or Google Cloud Load Balancing) are incredibly optimized for HTTP. They can multiplex and manage these connections much more efficiently.

### The Fan-Out Flow
* The Event: A data provider sends a "Wicket" event to your ingestion API.
* The Hub: Your API publishes a message to a Redis Topic (e.g., match_123).
* The Edge: 50 different server instances are subscribed to that Redis topic.
* The Push: Each of those 50 servers receives the message once and immediately iterates through its own local list of SSE connections, writing the data to the TCP socket.

### SSE + HTTP/2 = The Real Scalability Win
Before HTTP/2, if a user opened six tabs of your Cricinfo site, they would exhaust the browser's connection limit. With HTTP/2 (over HTTPS), all six of those SSE streams can happen over a single TCP connection.
