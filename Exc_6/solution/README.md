### Part 1

For this scenario, I would choose log-based CDC with Debezium. 
It reads committed changes from PostgreSQL's WAL and sends them to Kafka without changing the existing payment processor.

Polling would add unnecessary database load and could miss payments, 
while dual writes could lose events if the application crashes between the two writes.

The main disadvantage of CDC is that PostgreSQL has to keep WAL data if Debezium is not running or falls behind, 
which could eventually fill up the disk.
In production, I would monitor the replication slot lag, WAL size and available disk space to detect this problem early.

If our team owned the payment processor, I would prefer the transactional outbox pattern. 
It would allow us to store the payment and its corresponding event in the same database transaction.

### Part 2

**Run 1 – Default Configuration**

Prediction: I expect only one velocity agent to receive a partition, resulting in increasing lag and failing R4.

Results: R3: 2/7 frauds (29%, FAIL), 0 false alerts (PASS). R4: p95 latency 62.3 s (FAIL).

Explanation: Debezium created the topic with only one partition, so only one velocity agent could process payments. 
The agent could not keep up with 1,000 payments/s, causing increasing lag and missed frauds.

**Run 2 – More Partitions**

Prediction: I expect all three velocity agents to work in parallel, 
improving latency, but fraud detection will still fail because payments are partitioned by transaction ID.

Results: R3: 1/6 frauds (17%, FAIL), 0 false alerts (PASS). R4: p95 latency 0.6 s (PASS).

Explanation: With six partitions, all three agents could process payments in parallel. 
However, payments from the same cardholder were distributed across different agents, 
so no agent had the complete payment history.

**Run 3 – Cardholder as Key**

Prediction: Using user_id as the Kafka key should allow the velocity model to detect frauds correctly while keeping the low latency.

Results: R3: 6/6 frauds (100%, PASS), 0 false alerts (PASS). R4: p95 latency 1.2 s (PASS).

Explanation: All payments from the same cardholder now go to the same partition and agent. 
This allows the velocity model to maintain the complete state for each cardholder, 
while the six partitions still allow parallel processing.



