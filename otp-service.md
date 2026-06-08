## Design an OTP Service

### Functional Requrements
* Generate OTP: Create a secure, random N-digit token with a limited lifespan (e.g., 5 minutes).
* Deliver OTP: Send the token via SMS, Email, or WhatsApp.
* Verify OTP: Validate user-submitted tokens, handling success and failure states.

### Non-Functional Requirements:
* Low Latency: Generation and verification must happen in milliseconds <100ms
* High Availability: Auth paths are critical; if OTP is down, users can't log in.
* Security (Idempotency & Rate Limiting): Prevent brute-force attacks, SMS pumping (billing fraud), and token reuse.
* Cost Optimization: SMS gateways are expensive. The architecture must minimize unnecessary external API calls.


### High-Level Architecture
We leverage an in-memory data store for fast lookups and automatic expiration, combined with an asynchronous delivery pipeline to decouple token generation from external provider latency.

#### Key Components
1. API Gateway with Rate Limiter as first line of defense to restrict incoming requests per IP/User ID to prevent DDoS and API abuse before touching downstream services.
2. OTP Service: Stateless microservice which handles the core business logic. Generating a secure token, hashing them and storing in cache and triggering delivery
3. Distributed Cache: Stores the generate ORP against the user identity with TTL for automatic expiry
4. Message Queue: Kafka/Kinesis to decouple to generation from delivery. SMS/Email gateways can introduce latency spikes or temporary outages. The OTP Service should not be blocked by these outages
5. Notification Worker: consumes messages from the message queue and talks to external downstream APIs (twilio, Sendgrid, SNS, SES etc)


### Token Generation
1. We can use a cryptographically secure pseudo-random number generator -> Standard random() libray should never be used for this purpose
2. Instead of storing useId:token in Redis memory, we store 
```json
Key: "otp:user_id_or_phone"
Value: {
  "hashed_otp": SHA-256( "123456" + "user specific salt"),
  "attempts_left": 3,
  "generated_at": 1717843200
}
TTL: 300 seconds (5 minutes)
```

### Token Validation
Whe  client submits the token to the OTP server, it
1. Fetch Key from Redis
2. Check not exists, ie. Expiried / Missing -> Reject
3. Check `attempts_left` <= 0 -> Block & Delete Key
4. Hash Input Token == `hashed_otp`?
  1. YES: Return Success & Delete Key
  2. NO: Decrement `attempts_left`, Return Error

### Advanced Security & Production Considerations
### 1. Multi-Tiered Rate Limiting & Fraud Prevention
SMS pumping scams can drain thousands of dollars in hours. Implement a sliding-window rate limiter: 
* Per Phone Number/User: Max 1 OTP request per 60 seconds; max 5 requests per hour.
* Per IP Address: Max 20 requests per hour across all endpoints to prevent botnets from cycling through different phone numbers.

### 2. Handling SMS Gateway Failures (Circuit Breakers)
External APIs fail. If Twilio experiences an outage, your service shouldn't hang or drop requests completely.
  * Primary/Secondary Routing: Implement a circuit breaker pattern. If the primary provider fails consistently (>5% error rate), failover automatically to a secondary provider (e.g., Infobip or Vonage).
  * Idempotency Key: Generate a unique tracking ID for each notification job sent to the queue to ensure workers don't accidentally send duplicate SMS charges to a single user during network retries.

### 3. Time-Based One-Time Password (TOTP) Alternative
If your scale is massive and cost-prohibitive, consider offering TOTP (RFC 6238) via authenticator apps (Google Authenticator, Authy) for power users. This moves the computation and verification entirely to a mathematical formula based on a shared secret key, completely bypassing storage requirements and SMS gateway costs:

