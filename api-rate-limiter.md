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

1. Token Bucket (Most Popular):  

    **How It Works:**
    * A "bucket" holds tokens (requests).
    * Tokens are added at a fixed rate.
    * Each request consumes a token. If the bucket is empty, the request is denied.

    ```code
    Bucket capacity: 100 tokens
    Refill rate:     10 tokens/second
    Burst allowed:   Up to 100 simultaneous requests if bucket is full
    ```

    **Advantages:**
    * Smooth handling of bursts within limits.
    * Memory-efficient: only two values (token count, last refill timestamp) per client.
    * Widely used in production (e.g., AWS API Gateway, Stripe).

        
2. Leaky Bucket:  
   
    **How It Works:**
    * Requests are enqueued in a FIFO queue "the bucket"
    * Overflowing requests are rejected.
    * **A background process drains request** from a fixed `leaked_rate`

    ```code
    Queue size:   50 requests
    Leak rate:    5 requests/second
    Result:       Requests always exit at exactly 5/sec regardless of input
    ```

    **Advantages:**
    * Enforces a constant request rate, smoothing bursts.
    * Prevents upstream spikes from propagating.

    **Disadvantages:**
    * **Queued requests experience latency** even when the system is not overloaded.
    * A sudden legitimate burst will cause many requests to queue rather than be served immediately.
    * Extra processor for Queue management adds implementation complexity.
    
4. Fixed Window:
   
   **How It Works:**
    * Time is divided into fixed intervals (e.g., each minute from :00 to :59).
    * A counter tracks requests in the current window.
    * The counter resets when the window rolls over.

    ```code
    Window:   [12:00:00 – 12:59:59]  →  counter resets at 13:00:00
    Limit:    100 requests per window
    ```
    
    **Advantages:**
    * Simple to implement.
    * Low memory overhead: one counter per client per window.

    **Disadvantages:**
    * **Boundary burst vulnerability:** A client can send 100 requests at 12:59:59 and 100 more at 13:00:00, effectively doubling the limit within a two-second span.

5. Sliding Window:
    
    **How It Works:**
    * A timestamped log of every request is maintained per client.
    * On each incoming request, the system removes all entries older than the window duration and counts the remaining entries against the limit.
    * So unlike Fixed Window: Instead of a window that snaps to the clock, it's a window that follows the current moment. It always asks: "how many requests happened in the last 60 seconds?" — not "in this calendar minute."
      
    ```code
    Window:   60 seconds
    Limit:    100 requests
    Action:   At each request, evict entries with timestamp < (now - 60s), then count
    ```
    
    **Advantages:**
    * More accurate than fixed window for boundary issues.
  
   **Disadvantages**
   * High memory usage: stores one log entry per request per client.
   
