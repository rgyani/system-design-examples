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

    **Advantages:**
    * Smooth handling of bursts within limits.
    
2. Leaky Bucket:  
   
    **How It Works:**
    * Requests are enqueued in a "bucket."
    * The bucket processes requests at a fixed rate.
    * Overflowing requests are rejected.
   
    **Advantages:**
    * Enforces a constant request rate, smoothing bursts.
  
3. Fixed Window:
   
   **How It Works:**
    * Counts requests in fixed intervals (e.g., per minute).
    * Resets at the end of the interval.

    **Advantages:**
    * Simple to implement.

    **Disadvantages:**
    * Susceptible to bursts near window boundaries.

4. Sliding Window:
    
    **How It Works:**
    * Tracks requests using a rolling time window.
    * Counts requests in the last N seconds.
    
    **Advantages:**
    * More accurate than fixed window for boundary issues.