## Design a payment flow

### Requirement Gathering

User opens app -> enters amount -> selects bank -> request goes to backend  
backend validates user -> creates a transaction -> talks to payment gateway  
backend receives response from payment gateway -> updates transaction status -> shows success to the user

### What are the real problems here
1. What if user clicks pay button twice  --> We need idempotency, same request should not be processed twice
2. What if bank takes 30 seconds to respond --> we need timeout + state management + status polling, so we need to decouple this request
3. What if request to bank failed due to network connectivity -> We need to decouple payment request from UI
4. What if response from bank succeeds but our service crashes -> We need reconciliation jobs
5. What if notification failed -> Needs retry, so again decoupling
6. What happens if there are too many requests during a shopping spike period -> We need rate limiting, autoscaling
7. What happens when there is an audit requirement -> We need log, metrics and tracing


Now lets draw boxes

###  Step 1: The Ingestion & Idempotency Layer
1. User clicks pay. The client generates a unique idempotency_key (UUID).
2. API Gateway / Backend 1 receives the request.
3. Idempotency Check: It attempts to acquire a lock in Redis using idempotency_key.
4. If it exists, return the in-flight status or cached response.
5. If not, proceed to create Intent 
6. **Create a record in the Primary DB with status INITIATED and write a corresponding event to the Outbox Table in the same DB transaction.**
  * Same Transcation is critical here, since if one of the two requests fail, the transaction should be rolled back

### Step 2: Processing (The Async Handshake)
1. A CDC engine (like Debezium) reads the Outbox table and pushes the PaymentIntent to the Payment Request Stream.
2. Payment Executor (Workers) consumes the stream. It makes the external API call to the Payment Gateway.
3. The worker does not wait for a synchronous 30-second completion if the PG supports webhooks. It notes the transaction as PENDING_GATEWAY and yields.

### Step 3: Settlement & Reconciliation
1. The Payment Gateway sends an asynchronous webhook to your Webhook Gateway when the money actually clears.
2. The Webhook Gateway updates the Primary DB status to SUCCESS and appends rows to the Ledger Table (Debit/Credit).

### Step 4: Notification
1. Via CDC, a PaymentSuccess event hits the Response Stream, which Notification Service picks up to send a push notification/email, 
2. the Websocket Service uses to update the user's screen in real-time.

### Database
I would use relational database with strict ACID guarantees for the Ledger.   
Never use a pure NoSQL database for account balances unless it has robust multi-row transaction support.  
FYI, DynamodB does support 100 action request in a single transaction, across multuple tables 

### Double-Entry Ledger
In a payment system, **you can never rely on a single database column like user_balance = user_balance - 10.**   
If a database write fails halfway through, or a network timeout occurs, money can literally vanish or be created out of thin air.  

To prevent this, financial systems use **Double-Entry Bookkeeping.**  
The fundamental rule of double-entry ledger architecture is simple: 
Money cannot move without a source and a destination, and every single entry must balance to zero

#### The Core Principles
1. **No Overwriting/Updates:** You never run an UPDATE SQL command on an account balance. The ledger is an **append-only log** of immutable entries. 
2. If a mistake is made, **you don't edit the old row**; you write a new row to reverse it.
3. **Debits and Credits:** Every transaction consists of at least one Debit and at least one Credit.


### Audit
1. **Ingestion:** Kinesis Data Firehose captures events from your stream.
2. **Transformation & Parquet Format:** Before writing to S3, have Firehose convert the raw JSON into Apache Parquet or ORC format, and compress it using Snappy or GZIP.
   - Parquet is a columnar format. If you query a 10TB dataset looking only for transaction_id, Athena only scans that specific column, cutting your AWS costs by up to 90%.
3. **Partitioning:** Structure your S3 prefixes by date and event type:
   - s3://my-payment-lake/events/year=2026/month=06/day=06/payment_success/
4. **Querying (The Audit Power):** When compliance officers or accountants need to audit a specific spike from three months ago, they don't write code. They open Amazon Athena and run standard SQL directly against the files in S3:

#### Critical Security & Compliance Requirements
Because this is financial data, you can't just spin up a default S3 bucket.

**A. PCI-DSS Compliance & PII Masking**  
You cannot store raw credit card numbers (PANs) or CVVs anywhere, including S3.   
Before Firehose dumps data into the bucket, a lambda function or your upstream service must strip out or mask Personally Identifiable Information (PII) and financial secrets.

**B. Object Locking (WORM)**  
1. For strict financial regulations (like SEC rules or local banking laws), you must ensure that even a rogue administrator or a compromised AWS key cannot delete past transaction records.  
2. Enable S3 Object Lock in Compliance Mode (Write Once, Read Many). 
3. Once an event file is written to S3, it becomes physically un-deletable and un-modifiable for a set retention period (e.g., 7 years).

**C. Lifecycle Policies (Cost Optimization)**
Payment volume is massive, and storing everything on S3 Standard gets expensive. Implement an automated bucket lifecycle policy:
1. 0–90 Days: S3 Standard (Fast querying via Athena for recent support tickets).
2. 91–365 Days: S3 Standard-Infrequent Access (Standard-IA) or Glacier Instant Retrieval.
3. 1 Year+: Move to S3 Glacier Flexible or Deep Archive (Dirt cheap storage for long-term legal compliance where retrieval can take a few hours).

