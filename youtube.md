# Design YouTube

While it looks like Netflix on the surface, **the Upload Path is the real engineering monster here**. Netflix manages a few thousand high-quality files; YouTube manages billions of unpredictable ones.

## The High-Level Architecture
The system is split into three main "worlds":
* The Upload (Ingestion) Path: Where creators push raw files.
* The Metadata/Social Path: Where views, comments, and likes live.
* The Playback (Delivery) Path: Where users watch.

### The Ingestion Pipeline
Unlike Netflix, YouTube must handle massive write-volume from users.
* **Upload Processing:** Raw video goes to Blob Storage (GCS/S3). **A Message Queue** (Kafka) triggers the workers.
* **Transcoding & DAG:** YouTube uses a Directed Acyclic Graph (DAG) for processing. It doesn't just "convert" a file; it kicks off parallel tasks:
    - Extracting thumbnails.
    - Watermarking.
    - Generating multiple resolutions (144p to 8k).
    - Content ID Check: Running the file against a database of copyrighted audio/video.

### Data Storage & Scaling
YouTube uses a "Polyglot" approach to handle different data types:
- **BigTable (NoSQL):** Used for storing massive amounts of time-series data like **Video Metadata** and **User History.** It’s optimized for high-throughput writes.
- **Vitess (SQL on top of MySQL):** This was actually built by YouTube to scale their relational data. It handles **User Profiles** and **Subscriptions** — things that require more structure but still need to scale horizontally.
- **Redis/Memcached:** Crucial for "Hot" video metadata to avoid hitting the database every time a viral video is clicked.

### The Recommendation Engine (The "Brain")
This is the heart of YouTube’s "Watch Time" metric.
- **Two-Stage Approach:**
    1. **Candidate Generation:** Filters billions of videos down to hundreds based on user history and "collaborative filtering."
    2. **Ranking:** Uses a Deep Neural Network to score those hundreds against the user's current context (time of day, device, recent searches) to pick the top 10.
- **Tradeoff:** Freshness vs. Relevance. The system must balance showing you "New" videos with "Reliable" favorites.

### Key Tradeoffs
| Feature| Design Choice| The "Why"|
|---|---|---|
| Consistency| Eventual| View counts don't need to be 100% accurate every second. High availability is better than a perfect count.|
|Video Files| Chunking | Break 1GB into 5MB pieces. Easier to resume failed uploads and allows for Adaptive Bitrate Streaming.|
| Database| Vitess/Sharding| YouTube grew too big for standard MySQL; sharding by VideoID or UserID is mandatory.|
| Search| ElasticSearch/Solr| Needs fuzzy matching and "Inverted Indexes" to handle millions of queries per second.|
| Delivery| ISP Caching | Google Global Cache (GGC) nodes sit inside your local internet provider to save bandwidth.|

### Handling Viral Videos

Handling a viral video is the ultimate stress test for any system design. While a standard video might get a few hundred views, a viral one creates **"Hotspots"**—a massive spike in traffic directed at a single piece of data (the video file) and its metadata (view counts, likes, comments).

1. **The Playback Layer: Request Collapsing**  
When a video goes viral, you might get 100,000 requests for the same video segment at the exact same millisecond.
    - **The Problem:** If your server treats these as 100,000 individual tasks, it will crash.
    - **The Solution:** Use **Request Collapsing** (or "HTTP Request Joining") at the Edge/Proxy level. The proxy identifies that all these users want the same chunk, sends one request to the origin/cache, and multicasts the response back to all 100,000 users.
2. **The Cache Layer: Predictive "Thundering Herd" Prevention**  
Standard caching (LRU - Least Recently Used) is reactive. For virality, you need to be proactive.
    - **Edge Promotion:** As soon as a video's "velocity" (rate of change in views) hits a certain threshold, the system triggers a global replication.
    - **The Move:** The video is moved from standard regional storage to the **SSD-backed RAM caches of every single CDN node globally**, ensuring it never has to travel back to the "Home" server.
3. **The Metadata Layer: Adaptive Sharding**  
Viral videos often break databases because of the "View Count" problem. Writing to a single row in a database 1 million times a second causes lock contention
    - **In-Memory Buffering:** Don't write every view to the database immediately. Use a distributed counter in Redis.
    - **Sharded Counters:** Instead of one row for video_123_views, create 100 shards (rows) like video_123_views_1, video_123_views_2, etc. Increment a random shard for every view. When displaying the count, simply sum the shards ($Total = \sum_{i=1}^{n} Shard_i$).
4. **The Social Layer: Graceful Degradation**  
When the system is under extreme load from a viral event, you must prioritize Availability over Consistency.
    - **Circuit Breakers:** If the "Comments Service" is struggling to keep up with a viral video, the system can automatically trip a circuit breaker to hide comments or switch them to "Read Only."
    - **Prioritization:** Ensure that "Play Video" (the core product) gets all the resources, while non-essential features (like "Related Videos" or "Subtitles") are served from a stale cache or disabled temporarily.