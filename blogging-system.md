# Design a blogging system 

Design a simple multi-user blogging platform, allowing writers to publish and manage the blogs under their personal publication and readers to read them.

### Functional and Non-functional Requirements

Functional Requirements
1. writers should be able to publish a public blog (editing out of scope for now)
2. readers should be able to read the blog
3. blog posts should be searchable
4. a user can be both - a reader as well as a writer
5. The blog may contain text, images, videos
6. The writer/reader can see the blogs published for a user
7. Draft etc can be out-of-scope for now (but can be easily handled via a DB flag down the line, same for deletion/unpublish)
8. Comments are out-of-scope for now but will be good to discuss

Non-functional Requirements
1. Minimum latency
2. the platform should be scaled for 5 million daily active readers
3. the platform should be scaled for 10,000 daily active writers
4. Durable data, Strong consistency

### Why Not DynamoDB?
DynamoDB was considered for its schemaless flexibility and guaranteed constant-time performance at scale. However, it has a critical weakness for this use case: it has no native search capability. To support fuzzy/full-text search, you would still need a separate OpenSearch/Elasticsearch cluster on top — which negates the main advantage of going NoSQL.  
Additionally, a blogging platform has inherently relational data: users, posts, tags, and eventually comments, likes, and follows. DynamoDB forces rigid access pattern design upfront; a query like "all posts tagged 'python' by authors a user follows, sorted by likes" is natural in SQL and painful in DynamoDB.   
Finally, the write scale is simply not demanding enough to justify it — 10K daily active writers is a modest load (~0.12 writes/second average), well within what a managed PostgreSQL cluster handles effortlessly.

### Why Not a Separate Elasticsearch/OpenSearch Cluster?
A separate Elasticsearch cluster introduces significant operational complexity:
* Two systems to keep in sync — every write must go to both the DB and the search index; a failed Lambda trigger or indexing delay causes inconsistency
* Lambda cold starts add latency on every write path
* Indexing lag — OpenSearch is not instantly consistent, so freshly published posts may not appear in search results immediately
* Cost — running three separate AWS services (S3/DB + Lambda + OpenSearch) versus one

PostgreSQL's built-in full-text search (tsvector/tsquery) is sufficient for this scale. At 34 GB/year of text content (~10KB × 10K writers × 365 days), Postgres FTS handles millions of posts without breaking a sweat. A separate search cluster only becomes necessary beyond ~5–10M posts or when advanced requirements arise (semantic search, faceted filters, multi-language stemming at scale).

**Chosen Approach: PostgreSQL**

Data volume estimate: 10,000 writers/day × 365 days × 10KB per post ≈ 34 GB/year (text content only)  
This is comfortably handled by a single managed Postgres cluster with read replicas.

### Media (Images & Videos)
Text content lives in the DB. Binary media is stored separately:
* Images and videos → S3 bucket, served via CloudFront CDN
* The posts table stores S3 URLs/keys, not the files themselves
* Multi-resolution video variants (e.g., for different devices) can be handled by a background processor that writes new S3 keys; the posts table can be extended with additional columns (e.g., video_720p_url, video_1080p_url) without a schema migration headache, since adding nullable columns to Postgres is cheap

### Scaling & Read Performance
1M readers/month (~0.4 reads/second average, ~50–100 req/s at peak) does not stress the database — because reads should rarely hit the DB at all.

#### Read path:
Reader → CloudFront (CDN) → ElastiCache (Redis) → Aurora Postgres (cache miss only)

* CloudFront caches rendered blog pages at the edge; most reads never reach the origin
* Redis caches DB query results (e.g., post content, author page listings) with a short TTL
* DB only sees traffic on cache misses, keeping it nearly idle under normal load

#### Write path:
Writer → API → Aurora Postgres (content + search_vec updated atomically)
