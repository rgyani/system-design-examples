## Designing Instagram

### Functional Requirements
* Users can upload photos/videos
* Users can follow other users
* Users can view a feed of posts from people they follow
* Likes, comments, stories, DMs

### Non-functional:
* High availability (reads >> writes)
* Low latency feed loading (<200ms)
* Eventual consistency is acceptable


### Scale: 
Assume ~2B users, ~100M photos uploaded/day

**Capacity Estimation**
* Storage: 100M photos/day × 200KB avg = ~20TB/day
* Reads: Heavily read-dominant (~100:1 read/write ratio)
* DAU: ~500M users, each loading feed ~5x/day = 2.5B feed loads/day


### High-Level Architecture
```
Client → CDN → API Gateway → Microservices
                               ├── User Service
                               ├── Post Service
                               ├── Feed Service
                               ├── Media Service
                               └── Notification Service
```

### Key Components & Deep Dives

1. **Media Storage**
* Photos/videos stored in object storage (S3-like)
* Multiple resolutions generated on upload (thumbnail, medium, full)
* CDN (CloudFront/Akamai) for fast global delivery

2. **Database Design**
* Users: SQL (structured, relational, ACID needed)
* Posts/Follows: Cassandra or DynamoDB (high write throughput, scalable)
* **Feed cache: Redis (pre-computed feeds stored per user)**

3. **Feed Generation**
Two approaches:  

| | Fan-out on Write (Push)| Fan-out on Read (Pull)|
|---|---|--|
| How | On post, write to all followers' feeds| On load, fetch posts from all followees|
| Best for | Regular users| Celebrity/high-follower accounts|
| Tradeoff| Fast reads, slow/expensive writes| Slow reads, cheap writes|


Instagram, like twitter uses a hybrid: push for normal users, pull for celebrities (> ~1M followers).

### Search & Discovery
* Elasticsearch for hashtag/location search
* ML ranking for Explore page


3. Handling Scale
* Sharding: Shard user and post tables by user_id
* Replication: Read replicas for DB read scaling
* Rate limiting: Prevent abuse at API gateway
* Async processing: Message queues (Kafka) for feed updates, notifications, and media processing


### Infinite Scroll Feed Design
You can't load millions of posts at once. You need to fetch small chunks on demand, efficiently, without duplicates or gaps.

### Cursor-Based Pagination (the right approach)
Forget page numbers (?page=3). Instagram uses cursors.  
```GET /feed?cursor=<last_seen_post_id>&limit=12```

### Why cursors beat offset pagination:
||Offset (LIMIT 20 OFFSET 100) | Cursor-based |
|---|---|---|
|New posts inserted|Causes duplicates/skips|Handles gracefully|
|Deep pagination|Gets slower (DB scans all rows)|Constant time|
|Stateless|Yes|Yes|

The cursor is typically an **encoded timestamp + post_id combo**, making it both unique and sortable.

### Server Side: Pre-computed Feed Cache
When you scroll, the server isn't querying the DB live. It's reading from a pre-built feed stored in Redis:
```
user:feed:12345 → [postId_A, postId_B, postId_C, ... postId_N]
                    ↑ newest                              ↑ oldest
```

* Stored as a sorted set (score = timestamp)
* Typically capped at ~500–1000 most recent posts per user
* On scroll, you're just paginating through this Redis list — extremely fast

* **If the user scrolls past the cached window → fall back to DB query (rare).**

### Client Side: How It Triggers
```
Viewport
┌─────────────────┐
│   Post 1        │
│   Post 2        │
│   Post 3        │
│   Post 4        │  ← User scrolling down
│   Post 5        │
│   Post 6        │
│   Post 7        │
│                 │
│  [THRESHOLD]  ──│── Fire API call when user hits ~80% scroll depth
│                 │
│  (Loading...)   │
└─────────────────┘
```
* Client tracks scroll position with an Intersection Observer (not onScroll — too expensive)
* When the last visible post enters the viewport, fire the next fetch
* Append new posts to the existing list — never re-render the whole feed

### Deduplication
New posts can arrive while you're scrolling. Without handling this:
```
Initial load:    [A, B, C, D, E]
New post arrives: [Z, A, B, C, D, E]  ← Z inserted at top
Next page fetch:  [D, E, F, G]        ← D and E are duplicates!
```
Fix: The cursor anchors to a specific point in time (since_id). Posts newer than your session start are held separately and shown as "New posts — tap to refresh" rather than injected mid-scroll.


### Trade-offs
* Consistency vs. availability: Instagram favors availability — you may briefly see a stale feed
* Storage cost vs. latency: Pre-generating multiple image sizes costs storage but dramatically cuts load time
* Push vs. pull: No single answer; a hybrid is most practical at scale
