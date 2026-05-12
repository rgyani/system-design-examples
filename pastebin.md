# Design Pastebin

### Functional requirements
* Users can upload text (“pastes”)
* Generate short URL
* Retrieve paste by URL
* Optional expiration time
* Public/private/unlisted pastes

### Non-functional requirements
* High availability
* Low-latency reads
* Scalability
* Durability
* Security/moderation

### estimate scale.

* 10M pastes/day
* Average paste size = 10 KB
* Read-heavy system (maybe 100:1 reads:writes)

Quick storage estimate:
* 10M × 10 KB = 100 GB/day
* ~36 TB/year before replication/compression

### High-Level Architecture

Client
  |
Load Balancer
  |
API Servers
  |
+-------------------+
| Paste Service     |
| Auth Service      |
| Rate Limiter      |
+-------------------+
  |
Storage Layer
  |
+-------------------+
| Metadata DB       |
| Blob/Object Store |
| Cache (Redis)     |
+-------------------+


### Abuse & Security

Paste services get abused constantly.
We should implement
* rate limiting
* spam detection
* moderation pipelines
* max paste size