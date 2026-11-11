# Setup

I did Part 0 and Part 1 as instructed and ran into no troubles. 

# Part A
### Step 1

````bash
run 1794405141: 20 machines x 10 Hz = 200 readings/s -> postgres (Ctrl+C to stop)
  t=    5s      1,020 readings sent (204/s)
  t=   10s      2,020 readings sent (202/s)
  t=   15s      3,020 readings sent (201/s)
  t=   20s      4,020 readings sent (201/s)
  t=   25s      5,020 readings sent (201/s)
  t=   30s      6,020 readings sent (201/s)
  t=   35s      7,020 readings sent (201/s)
  t=   40s  fault starts on M-006 (repaired in 45 s)
  t=   40s      8,020 readings sent (200/s)  faulty: M-006
  t=   45s      9,020 readings sent (200/s)  faulty: M-006
  t=   50s     10,020 readings sent (200/s)  faulty: M-006
  t=   55s     11,020 readings sent (200/s)  faulty: M-006
  t=   60s     12,020 readings sent (200/s)  faulty: M-006
  t=   65s     13,020 readings sent (200/s)  faulty: M-006
  t=   70s     14,020 readings sent (200/s)  faulty: M-006
  t=   75s     15,020 readings sent (200/s)  faulty: M-006
  t=   80s     16,020 readings sent (200/s)  faulty: M-006
  t=   85s     17,020 readings sent (200/s)  faulty: M-006
  t=   90s     18,020 readings sent (200/s)
  t=   95s     19,020 readings sent (200/s)
^Crun 1794405141: sent 19,720 readings in 99 s
````

The plant sent 19,720 readings, and all of them were stored in PostgreSQL. 
But, the detector only processed 16,350 readings and reported 3,366 missing readings, 
even though the backlog reached 0. 

In total, 3,370 readings were not processed, while the model reported 3,366 missing readings.

The problem is caused by the way PostgreSQL handles transactions. 
IDs are assigned during INSERT, but rows only become visible after COMMIT. 
Therefore, a row with a higher ID might become visible before a row with a lower ID.

Because the detector only gets rows with id > last_id, it can skip rows that are committed later. 
This explains why readings are missing even though they exist in the database.

### Step 2

````bash
(venv) langs@LangiLaptop:/mnt/c/Users/langs/Documents/hagenberg/s3/sbd/sbd-ais-ws26/Exc_5/guardian$ python3 plant.py --sink postgres
run 1794405884: 60 machines x 10 Hz = 600 readings/s -> postgres (Ctrl+C to stop)
  t=    5s      3,060 readings sent (612/s)
  t=   10s      6,060 readings sent (606/s)
  t=   15s      9,060 readings sent (604/s)
  t=   20s     12,060 readings sent (603/s)
  t=   25s     15,060 readings sent (602/s)
  t=   30s     18,060 readings sent (602/s)
  t=   35s     21,060 readings sent (602/s)
  t=   40s  fault starts on M-029 (repaired in 45 s)
  t=   40s     24,060 readings sent (601/s)  faulty: M-029
  t=   45s     27,060 readings sent (601/s)  faulty: M-029
  t=   50s     30,060 readings sent (601/s)  faulty: M-029
  t=   55s     33,060 readings sent (601/s)  faulty: M-029
  t=   60s     36,060 readings sent (601/s)  faulty: M-029
^Crun 1794405884: sent 37,500 readings in 63 s
````

With 60 machines producing 600 readings/s, the detector was too slow. 
The backlog increased to almost 18,000 readings, while the delay grew from 2.8 s to over 33 s.

The main reason is the models inference time of 3 ms per reading. 
A single detector can theoretically process only about 333 readings/s, 
which is much less than the 600 readings/s produced by the plant.

The number of missing readings stayed at 30, 
but the increasing backlog shows that many readings were still not processed. 
No anomalies were detected before stopping the detector because it had not yet reached the faulty readings.

This means the current system cannot meet the requirement of detecting anomalies within 5 seconds at full load.



# Part B

### Option 1: PostgreSQL with multiple pollers

Using multiple pollers could improve performance by distributing the machines across several detector instances. 
But, the problem with missing readings due to transaction commit order would still need to be solved. 
More consumers, like a dashboard or training archive, would also increase the load on PostgreSQL.

### Option 2: Kafka

Kafka is better suited for continuous streams of sensor data. 
By using the machine ID as the message key, all readings of one machine can be sent to the same partition, 
preserving their order. Multiple detector instances can process different partitions in parallel. 
Independent consumer groups also allow the dashboard and archive to read the same data without interfering with the detector.

### Decision
I would choose Kafka for processing the sensor readings. 

Kafka supports scaling through partitions and consumer groups (R1), 
preserves per-machine ordering when using machine IDs as keys (R2), 
and allows independent consumers without adding more PostgreSQL load (R4). 
With enough detectors, the required latency of 5 seconds (R3) should also be possible.

### Required detector instances:

The model needs 3 ms per reading, so one detector can process 333 readings/s (max).

- 600 readings/s: at least 2 detector instances
- 100,000 readings/s: at least 300 detector instances

These are theoretical minimums. In practice, I would use more instances to handle overhead and load spikes.

### Tipping point:

For a small system with only a few readings per minute, PostgreSQL is sufficient. 
Kafka becomes more useful when the system needs to process hundreds or thousands 
of readings per second with low latency and multiple independent consumers. 
The experiment already showed performance problems at 600 readings/s.