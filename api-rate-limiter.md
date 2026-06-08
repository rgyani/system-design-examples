# Design API Rate Limiter

A Rate Limiter limits the number of client requests allowed to be sent over a specified period. If the API request count exceeds the threshold defined by the rate limiter, all the excess calls are blocked.

Benefits of rate-limiter
* A rate limiter prevents DoS attacks, intentional or unintentional, by blocking the excess calls.
* Reduces cost where the system is using a 3rd-party API service and is charged on a per-call-basis.
* To reduce server load, a rate limiter is used to filter out excess requests caused by bots or users’ misbehaviour.


## 1. Define Requirements
   * Granularity: Decide whether limits will be per user, per IP, or per API key.
   * Limits: Specify thresholds (e.g., 100 requests per minute).
   * Behavior on Limit Exceed: Define what happens when limits are exceeded (e.g., reject with HTTP 429 Too Many Requests).
   * Global Limits: Consider a global rate limit for the entire API if needed.
  
## 2. Choose an Algorithm
Several algorithms are commonly used for rate limiting:

## Token Bucket (Most Popular):  

#### How It Works:
* A "bucket" holds tokens (requests).
* Tokens are added at a fixed rate.
* Each request consumes a token. If the bucket is empty, the request is denied.

```code
Bucket capacity: 100 tokens
Refill rate:     10 tokens/second
Burst allowed:   Up to 100 simultaneous requests if bucket is full
```

#### Advantages:
* Smooth handling of bursts within limits.
* Memory-efficient: only two values (token count, last refill timestamp) per client.
* Widely used in production (e.g., AWS API Gateway, Stripe).

#### Practical Implementation: 
* Our mental model envisons a backend service filling the bucket (an in-memory) with tokens per user is an operation nightmare, and not practical.
* Instead the API/backend itself stores in REDIS two numbers for each user
    - last_refill_time: A timestamp of exactly when that user last made a request.
    - tokens: The number of tokens left in their bucket at that specific moment.
* When a request hits the rate-limiting backend, it fetches these values from in-memory store and 
    - calculate the delta: **time_passed = current_time - last_refill_time**
    - virtual refills the tokens: **new_tokens = tokens + time_passed * refill_rate**
    - calculates tokens available : **tokens = min(max_capacity, new_tokens)**
    - If **tokens >= 1**: It decrements the token count by 1, updates Redis with the new token count and the new last_refill_time, and forwards the request to your core backend services.
    - Else: returns 429: Too many Requests


## Leaky Bucket:  
   
#### How It Works:
* Requests are enqueued in a FIFO queue "the bucket"
* Overflowing requests are rejected.
* **A background process drains request** from a fixed `leaked_rate`

```code
Queue size:   50 requests
Leak rate:    5 requests/second
Result:       Requests always exit at exactly 5/sec regardless of input
```

#### Advantages:
* Enforces a constant request rate, smoothing bursts.
* Prevents upstream spikes from propagating.

#### Disadvantages:
* **Queued requests experience latency** even when the system is not overloaded.
* A sudden legitimate burst will cause many requests to queue rather than be served immediately.
* Extra processor for Queue management adds implementation complexity.
    
### Practical Implementation
* In a production-grade, high-throughput system, we do not enqueue the actual requests into Redis, and we do not use a background process to poll them.
* Instead, we use Virtual Leaking, which is entirely mathematical.
    * Instead of a queue of requests, Redis only stores one single number for a user: **next_allowed_time**

Suppose A Sudden Burst of 60 Requests Arrives at Once
* Let's call this time T = 0.0s
* Here is exactly how the API server processes them, one by one, using quick math against Redis:
* The 1st Request Arrives
    - Redis is empty, so next_allowed_time is 0.0s. 
    - This request can go through immediately.
    -  The server calculates when this request will "finish leaking." Since the rate is 1 request per 200ms, it takes 200ms to process.
    - It updates next_allowed_time to 0.2s.
    - Result: ACCEPTED. (Sent to your backend immediately).
* The 2nd Request Arrives (at the same millisecond)
    - The server checks Redis and sees the virtual queue is busy until 0.2s. 
    - So, this 2nd request is virtually placed right behind it.
    - It will finish leaking 200ms later, at 0.4s
    - Updates next_allowed_time to 0.4s
    - Result: ACCEPTED.
* The 50th Request Arrives (at the same millisecond)
    - Following the same pattern, the 50th request gets stacked at the end of the line.
    - Redis update: 50 * 200m = 10,000ms = 10 sec
    - The virtual queue is now backed up for the next 10 seconds.
    - Our max queue size is 50 requests. At 5 requests/sec, a full queue takes exactly 10 seconds to drain. Because our next_allowed_time 10s does not exceed our maximum allowed backup 10s, it barely fits!
    - Updates next_allowed_time to 10.0s
    - Result: ACCEPTED.
* The 51st Request Arrives (at the same millisecond)
    - The server checks Redis. The virtual queue is busy until 10.0s. 
    - Stacking this request would push the exit time to 10.2s
    - The server calculates the total backup: 10.2s - current time (0.0s) = 10.2s
    - 10.2s is greater than our maximum allowed backup of 10s
    - Result: REJECTED (HTTP 429 Too Many Requests).
* Requests 52 through 60 will face the exact same fate and get rejected immediately.

* A new request comes at T = 0.5s
    - The server fetches next_allowed_time from Redis, which is still sitting at 10.0s
    - Calculate Current Backup: 10.0s - current time (0.5s) = 9.5s
    - Since 9.5s < 10s? Yes! Because 0.5s passed, the virtual bucket "leaked" enough space to accept exactly 2.5 requests.
    - Stacking this new request onto the queue pushes the time from 10.0s to 10.2s
    - Updates next_allowed_time to 10.2s
    - Result: ACCEPTED.


### Fixed Window:

#### How It Works
* Time is divided into fixed intervals (e.g., each minute from :00 to :59).
* A counter tracks requests in the current window.
* The counter resets when the window rolls over.

```code
Window:   [12:00:00 – 12:59:59]  →  counter resets at 13:00:00
Limit:    100 requests per window
```
    
#### Advantages:
* Simple to implement.
* Low memory overhead: one counter per client per window.

#### Disadvantages
* **Boundary burst vulnerability:** A client can send 100 requests at 12:59:59 and 100 more at 13:00:00, effectively doubling the limit within a two-second span.

### Sliding Window:
    
#### How It Works
* A timestamped log of every request is maintained per client.
* On each incoming request, the system removes all entries older than the window duration and counts the remaining entries against the limit.
* So unlike Fixed Window: Instead of a window that snaps to the clock, it's a window that follows the current moment. It always asks: "how many requests happened in the last 60 seconds?" — not "in this calendar minute."
      
```code
Window:   60 seconds
Limit:    100 requests
Action:   At each request, evict entries with timestamp < (now - 60s), then count
```
    
#### Advantages
* More accurate than fixed window for boundary issues.
  
#### Disadvantages
* High memory usage: stores one log entry per request per client.
   
