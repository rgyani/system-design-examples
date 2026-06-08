## Design a Distributed Job Schedular

The job state in the database drives the orchestration. Here is how the lifecycle looks as it moves through our architecture:

```
[API Gateway] ──> (DB: PENDING)
                       │
                       ▼
             [Scheduler Leader Polls]
                       │
                       ▼
         (DB: QUEUED) ──> [ SQS Queue ] 
                               │
                               ▼
                        (Queue Depth Trigger) 
                               │
                               ▼
                        [Workers Scale Up]
                               │
                               ▼
                           (DB: RUNNING)
                               │
          ┌────────────────────┴────────────────────┐
          ▼                                         ▼
      [Job Succeeds]                            [Job Fails]
          │                                         │
          ▼                                         ▼
     (DB: SUCCESS)                            [ SQS DLQ Trigger ]
                                                    │
                                                    ▼
                                               [DLQ Processor]
                                                    │
                                                    ▼
                                               (DB: FAILED)

```


### Scheduling Phase
1. API Gateway: Validates the request, and stored the Job MetaData is a DB
2. Scheduler Leader: We have multiple scheduler nodes with a leader election polling the DB. It prevents a "race condition" where two schedulers pull the exact same pending job and push it to SQS twice.

### Execution Phase
Triggering worker scaling based on SQS Queue Depth (e.g., using AWS Auto Scaling Policies based on ApproximateNumberOfMessagesVisible) is a cloud-native best practice.
1. **Visibility Timeout:** When a worker picks up a message from SQS, the message doesn't disappear immediately; it becomes invisible to other workers for a set "Visibility Timeout."
2. **Setting to RUNNING:** The moment the worker pulls the message, its first API call should be back to the Metadata DB to change the status from QUEUED to RUNNING.
3. **Long Running Jons**: Ensure your workers use a "Heartbeat/Visibility Extension" mechanism. While the job is running, the worker should periodically call SQS ChangeMessageVisibility to tell the queue, "I'm still working on this, give me 5 more minutes."

### The Failure Phase (DLQ & Processor)
Using the SQS Dead Letter Queue (DLQ) native retry behavior (e.g., retry 3 times, then move to DLQ) is highly elegant.
1. **The DLQ Processor:** Having a dedicated consumer watching the DLQ to mark jobs as FAILED in the DB is perfect. It keeps your worker code clean, as the workers don't need complex catch-blocks for infrastructure crashes; if the worker container vaporizes mid-job, SQS handles the timeout, pushes it to the DLQ, and your processor updates the DB.

## Problems

### 1. The Polling Bottleneck & "The Thundering Herd"
The Question: "Your scheduler leader is polling the DB for PENDING jobs. What happens if there are 50 million pending jobs due at exactly midnight? Your database CPU spikes to 100%, queries time out, and the scheduler falls behind. How do you fix the database bottleneck?"

* **Time-Bucketing & Partitioning:** We can partition or shard our jobs table by a scheduled_time_bucket (e.g., hourly buckets). The schedulers only query the current bucket, avoiding a full table scan.
* **The Redis Hand-off:** Move the immediate-execution window out of the relational DB. The scheduler leader grabs jobs due in the next 10 minutes and pushes their IDs into a Redis Sorted Set (ZSET), where the score is the Unix timestamp. Schedulers then query Redis (which handles sub-millisecond sorted lookups) to feed SQS, completely offloading the relational DB during peak spikes.

### 2. The SQS Scale-Up Lag
The Question: "You are scaling workers based on SQS queue depth. If 10,000 jobs drop into SQS instantly, AWS Auto Scaling has a cold-start lag. It takes minutes to provision containers/VMs. During those minutes, jobs are sitting idle. If these are low-latency jobs, your system is failing its SLA. How do you mitigate this?"
 * **Predictive Pre-warming:** If the scheduler leader already knows a massive batch of jobs is due at 12:00 AM (because it can see the recurring cron metadata in the DB), it can emit an alert or invoke an AWS SDK command to step up the worker pool's minimum capacity 5 minutes before midnight.
* **Target Tracking vs. Step Scaling:** Instead of using simple step scaling, we can use Target Tracking Scaling based on a custom metric like Backlog Per Instance (Queue Depth / Number of active workers) to make the scaling curve aggressive enough to match heavy ingestion.

### 3. The "Two Leaders" Split-Brain
The Question: "You are using leader election for your schedulers. There is a network partition. Scheduler A is the leader, but it gets disconnected from the rest of the cluster. The remaining nodes elect Scheduler B as the new leader. Now you have two leaders. What stops both from polling the same PENDING jobs and pushing duplicates into SQS?"
* Use a centralized Leader Election like Zookeeper or even DB locks

### 4. Why using SQS, why not Kafka/Kinesis
**Operational Overhead**. Kafka requires partition management, consumer group rebalancing, and complex offset tracking to scale workers dynamically. SQS gives you out-of-the-box, per-message visibility tracking, dead-letter routing, and automatic scaling hooks—making it a far lighter and more maintainable architecture for a pure task-distribution pipeline.

Further Reading: https://www.youtube.com/watch?v=WTxG5880EH8