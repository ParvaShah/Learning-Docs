# Kafka Architecture: Complete Component Guide
## From 30,000 Feet to Ground Level

**Table of Contents:**
1. [Level 1: The Kafka Ecosystem (30,000 ft view)](#level-1-the-kafka-ecosystem)
2. [Level 2: Kafka Cluster Architecture](#level-2-kafka-cluster-architecture)
3. [Level 3: Broker Internals](#level-3-broker-internals)
4. [Level 4: Topic Structure](#level-4-topic-structure)
5. [Level 5: Partition Deep Dive](#level-5-partition-deep-dive)
6. [Level 6: Segment Files & Storage](#level-6-segment-files--storage)
7. [Level 7: Producer Architecture](#level-7-producer-architecture)
8. [Level 8: Consumer Architecture](#level-8-consumer-architecture)
9. [Level 9: Request Flow & Protocols](#level-9-request-flow--protocols)
10. [Level 10: Replication & High Availability](#level-10-replication--high-availability)
11. [Common Conversation Topics](#common-conversation-topics)

---

## Level 1: The Kafka Ecosystem (30,000 ft view)

### The Big Picture

```
┌─────────────────────────────────────────────────────────────────┐
│                    KAFKA ECOSYSTEM                              │
│                                                                 │
│  ┌─────────────┐         ┌──────────────┐      ┌────────────┐ │
│  │  PRODUCERS  │────────▶│    KAFKA     │─────▶│  CONSUMERS │ │
│  │             │         │   CLUSTER    │      │            │ │
│  │ - Apps      │         │              │      │ - Apps     │ │
│  │ - Services  │         │  (Brokers)   │      │ - Services │ │
│  │ - Logs      │         │              │      │ - DBs      │ │
│  └─────────────┘         └──────────────┘      └────────────┘ │
│                                 │                               │
│                                 │                               │
│                          ┌──────▼──────┐                       │
│                          │  ZooKeeper  │                       │
│                          │  or KRaft   │                       │
│                          │ (Metadata)  │                       │
│                          └─────────────┘                       │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │              OPTIONAL COMPONENTS                          │ │
│  │                                                            │ │
│  │  - Kafka Connect (Data Integration)                       │ │
│  │  - Kafka Streams (Stream Processing)                      │ │
│  │  - Schema Registry (Schema Management)                    │ │
│  │  - ksqlDB (SQL on Streams)                                │ │
│  │  - REST Proxy (HTTP API)                                  │ │
│  └──────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Key Concepts at This Level:

**Kafka Cluster:**
- The core distributed messaging system
- Stores and serves data
- Made up of multiple brokers

**Producers:**
- Write data TO Kafka
- Push model (producers push)
- Any application that sends messages

**Consumers:**
- Read data FROM Kafka
- Pull model (consumers pull)
- Any application that reads messages

**ZooKeeper/KRaft:**
- Metadata management
- Cluster coordination
- Leader election
- Modern Kafka moving to KRaft (removing ZooKeeper dependency)

---

## Level 2: Kafka Cluster Architecture

### Cluster Components

```
┌─────────────────────────────────────────────────────────────────────┐
│                         KAFKA CLUSTER                               │
│                                                                     │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐ │
│  │   BROKER 1       │  │   BROKER 2       │  │   BROKER 3       │ │
│  │   (broker.id=1)  │  │   (broker.id=2)  │  │   (broker.id=3)  │ │
│  │                  │  │                  │  │                  │ │
│  │  Port: 9092      │  │  Port: 9092      │  │  Port: 9092      │ │
│  │  Host: kafka-1   │  │  Host: kafka-2   │  │  Host: kafka-3   │ │
│  │                  │  │                  │  │                  │ │
│  │  ┌────────────┐  │  │  ┌────────────┐  │  │  ┌────────────┐  │ │
│  │  │  Topics    │  │  │  │  Topics    │  │  │  │  Topics    │  │ │
│  │  │ Partitions │  │  │  │ Partitions │  │  │  │ Partitions │  │ │
│  │  └────────────┘  │  │  └────────────┘  │  │  └────────────┘  │ │
│  │                  │  │                  │  │                  │ │
│  │  Disk Storage    │  │  Disk Storage    │  │  Disk Storage    │ │
│  │  /var/kafka/data │  │  /var/kafka/data │  │  /var/kafka/data │ │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘ │
│           │                     │                     │            │
│           └─────────────────────┼─────────────────────┘            │
│                                 │                                  │
└─────────────────────────────────┼──────────────────────────────────┘
                                  │
                                  │
                    ┌─────────────▼─────────────┐
                    │   COORDINATION LAYER      │
                    │                           │
                    │  ZooKeeper Ensemble       │
                    │  ┌──────┐  ┌──────┐      │
                    │  │ ZK 1 │  │ ZK 2 │      │
                    │  └──────┘  └──────┘      │
                    │       ┌──────┐           │
                    │       │ ZK 3 │           │
                    │       └──────┘           │
                    │                           │
                    │  OR                       │
                    │                           │
                    │  KRaft (Kafka Raft)       │
                    │  - Controller nodes       │
                    │  - No external dependency │
                    └───────────────────────────┘
```

### What Each Component Does:

**Broker:**
- A single Kafka server instance
- Stores data on disk
- Serves produce and fetch requests
- Manages partitions
- Replicates data

**Cluster:**
- Collection of brokers working together
- Appears as one logical system
- Provides scalability and fault tolerance

**Controller:**
- ONE broker elected as cluster controller
- Manages partition assignments
- Handles broker failures
- Coordinates leader elections

**ZooKeeper (Legacy):**
```
Stores:
  - Broker metadata
  - Topic configurations
  - Partition assignments
  - Controller election
  - ACLs

Quorum: 3 or 5 nodes (odd number)
Purpose: Distributed coordination
```

**KRaft (New, Kafka 3.0+):**
```
Benefits:
  - No external dependency
  - Faster metadata operations
  - Simpler deployment
  - Better scalability

Implementation:
  - Metadata stored in Kafka itself
  - Uses Raft consensus protocol
  - Controller nodes form quorum
```

---

## Level 3: Broker Internals

### What's Inside a Broker

```
┌─────────────────────────────────────────────────────────────────┐
│                      KAFKA BROKER                               │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐│
│  │                    NETWORK LAYER                           ││
│  │                                                            ││
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   ││
│  │  │   Acceptor   │  │  Processor   │  │  Processor   │   ││
│  │  │   Thread     │─▶│  Thread 1    │  │  Thread 2    │   ││
│  │  │              │  │              │  │              │   ││
│  │  │  Port: 9092  │  │  Handle      │  │  Handle      │   ││
│  │  │              │  │  Requests    │  │  Requests    │   ││
│  │  └──────────────┘  └──────────────┘  └──────────────┘   ││
│  │                                                            ││
│  └────────────────────────────────────────────────────────────┘│
│                                │                                │
│                                ▼                                │
│  ┌────────────────────────────────────────────────────────────┐│
│  │                    REQUEST QUEUE                           ││
│  │  [Req1] [Req2] [Req3] [Req4] [Req5] ...                   ││
│  └────────────────────────────────────────────────────────────┘│
│                                │                                │
│                                ▼                                │
│  ┌────────────────────────────────────────────────────────────┐│
│  │                   API HANDLER THREADS                      ││
│  │                                                            ││
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐               ││
│  │  │ Handler  │  │ Handler  │  │ Handler  │               ││
│  │  │ Thread 1 │  │ Thread 2 │  │ Thread 3 │  ...          ││
│  │  └──────────┘  └──────────┘  └──────────┘               ││
│  │       │             │             │                       ││
│  └───────┼─────────────┼─────────────┼───────────────────────┘│
│          │             │             │                        │
│          ▼             ▼             ▼                        │
│  ┌────────────────────────────────────────────────────────────┐│
│  │                  LOG MANAGER                               ││
│  │                                                            ││
│  │  Manages:                                                  ││
│  │  - Topic creation/deletion                                 ││
│  │  - Partition assignment                                    ││
│  │  - Log segment lifecycle                                   ││
│  │  - Log cleaning (compaction)                               ││
│  │  - Log retention                                           ││
│  │                                                            ││
│  │  ┌──────────────────────────────────────────────────────┐ ││
│  │  │             LOG CLEANER MANAGER                      │ ││
│  │  │  ┌────────┐  ┌────────┐  ┌────────┐                │ ││
│  │  │  │Cleaner │  │Cleaner │  │Cleaner │                │ ││
│  │  │  │Thread 1│  │Thread 2│  │Thread 3│                │ ││
│  │  │  └────────┘  └────────┘  └────────┘                │ ││
│  │  │  (Handles compaction for compacted topics)          │ ││
│  │  └──────────────────────────────────────────────────────┘ ││
│  │                                                            ││
│  └────────────────────────────────────────────────────────────┘│
│                                │                                │
│                                ▼                                │
│  ┌────────────────────────────────────────────────────────────┐│
│  │                 REPLICA MANAGER                            ││
│  │                                                            ││
│  │  - Manages partition replicas on this broker              ││
│  │  - Handles replication from leader                        ││
│  │  - Coordinates follower fetching                          ││
│  │  - Manages ISR (In-Sync Replicas)                         ││
│  │                                                            ││
│  │  ┌──────────────────────────────────────────────────────┐ ││
│  │  │           REPLICATION FETCHER THREADS                │ ││
│  │  │  (For partitions where this broker is follower)      │ ││
│  │  │  ┌────────┐  ┌────────┐  ┌────────┐                │ ││
│  │  │  │Fetcher │  │Fetcher │  │Fetcher │                │ ││
│  │  │  │   1    │  │   2    │  │   3    │                │ ││
│  │  │  └────────┘  └────────┘  └────────┘                │ ││
│  │  └──────────────────────────────────────────────────────┘ ││
│  └────────────────────────────────────────────────────────────┘│
│                                │                                │
│                                ▼                                │
│  ┌────────────────────────────────────────────────────────────┐│
│  │                   DISK STORAGE                             ││
│  │                                                            ││
│  │  /var/kafka/data/                                          ││
│  │    ├── topic-A-0/     (Partition 0 of topic-A)            ││
│  │    ├── topic-A-1/     (Partition 1 of topic-A)            ││
│  │    ├── topic-B-0/     (Partition 0 of topic-B)            ││
│  │    └── topic-B-1/     (Partition 1 of topic-B)            ││
│  │                                                            ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐│
│  │                  OTHER COMPONENTS                          ││
│  │                                                            ││
│  │  - Group Coordinator (manages consumer groups)            ││
│  │  - Transaction Coordinator (handles transactions)         ││
│  │  - Metrics Reporter (JMX metrics)                         ││
│  │  - Admin Manager (admin operations)                       ││
│  │  - Security Manager (authentication/authorization)        ││
│  └────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### Thread Model Explained:

**Acceptor Thread:**
```
Role: Accepts new TCP connections
Count: 1 per endpoint (1 for PLAINTEXT, 1 for SSL, etc.)
Action: Hands off connections to processor threads
```

**Processor Threads (Network Threads):**
```
Role: Handle network I/O (read requests, send responses)
Count: Configurable (num.network.threads, default 3)
Action: Read from socket → Place in request queue
        Take from response queue → Write to socket
Non-blocking: Uses Java NIO (selectors)
```

**API Handler Threads (I/O Threads):**
```
Role: Process actual requests (produce, fetch, metadata, etc.)
Count: Configurable (num.io.threads, default 8)
Action: Take from request queue → Process → Place in response queue
Blocking: Can block on disk I/O
```

**Request Flow:**
```
1. Client connects → Acceptor Thread
2. Acceptor assigns to → Processor Thread
3. Processor reads request → Request Queue
4. Handler Thread picks up → Processes request
5. Handler places response → Response Queue
6. Processor writes back → Client
```

---

## Level 4: Topic Structure

### Topic Organization

```
┌─────────────────────────────────────────────────────────────────┐
│                         TOPIC: "orders"                         │
│                                                                 │
│  Configuration:                                                 │
│  - partitions: 3                                                │
│  - replication.factor: 3                                        │
│  - cleanup.policy: delete                                       │
│  - retention.ms: 604800000 (7 days)                             │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                    PARTITION 0                            │ │
│  │  Leader: Broker 1                                         │ │
│  │  Replicas: [Broker 1, Broker 2, Broker 3]                │ │
│  │  ISR: [Broker 1, Broker 2, Broker 3]                     │ │
│  │                                                           │ │
│  │  Offsets: 0 ────────────────────────────────▶ 1,234,567  │ │
│  │           [========= Messages ===========]                │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                    PARTITION 1                            │ │
│  │  Leader: Broker 2                                         │ │
│  │  Replicas: [Broker 2, Broker 3, Broker 1]                │ │
│  │  ISR: [Broker 2, Broker 3, Broker 1]                     │ │
│  │                                                           │ │
│  │  Offsets: 0 ────────────────────────────────▶ 987,654    │ │
│  │           [========= Messages ===========]                │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                    PARTITION 2                            │ │
│  │  Leader: Broker 3                                         │ │
│  │  Replicas: [Broker 3, Broker 1, Broker 2]                │ │
│  │  ISR: [Broker 3, Broker 1, Broker 2]                     │ │
│  │                                                           │ │
│  │  Offsets: 0 ────────────────────────────────▶ 1,111,111  │ │
│  │           [========= Messages ===========]                │ │
│  └───────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘

HOW MESSAGES ARE DISTRIBUTED:

Producer sends message with key "customer-123":
  ↓
Hash(key) % num_partitions = target partition
  ↓
hash("customer-123") % 3 = 2
  ↓
Message goes to Partition 2

Producer sends message with no key:
  ↓
Round-robin or sticky partitioning
  ↓
Messages distributed evenly across partitions
```

### Key Topic Concepts:

**Partition:**
- Ordered, immutable sequence of messages
- Each message gets sequential ID (offset)
- Unit of parallelism

**Replication:**
- Each partition has multiple copies
- Copies spread across different brokers
- Provides fault tolerance

**Leader:**
- One replica designated as leader
- All reads and writes go to leader
- Leader manages replicas

**Follower:**
- Other replicas are followers
- Passively replicate from leader
- Can become leader if current leader fails

**ISR (In-Sync Replicas):**
- Set of replicas that are caught up with leader
- Includes leader + followers that are not "too far behind"
- Only ISR members can become leader

---

## Level 5: Partition Deep Dive

### Inside a Partition

```
┌─────────────────────────────────────────────────────────────────┐
│                PARTITION: orders-0 (on Broker 1)                │
│                                                                 │
│  Metadata:                                                      │
│  - Topic: orders                                                │
│  - Partition ID: 0                                              │
│  - Leader: Broker 1 (THIS BROKER)                               │
│  - ISR: [1, 2, 3]                                               │
│  - High Water Mark (HWM): 1,234,567                             │
│  - Log End Offset (LEO): 1,234,570                              │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                    ACTIVE SEGMENT                         │ │
│  │                                                           │ │
│  │  00000000001230000.log   (currently being written)       │ │
│  │  00000000001230000.index (offset → position)             │ │
│  │  00000000001230000.timeindex (timestamp → offset)        │ │
│  │                                                           │ │
│  │  Size: 512 MB (growing)                                   │ │
│  │  Base Offset: 1,230,000                                   │ │
│  │  Current Offset: 1,234,570                                │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                   CLOSED SEGMENTS                         │ │
│  │                                                           │ │
│  │  00000000000000000.log (segment 1)                        │ │
│  │  00000000000000000.index                                  │ │
│  │  00000000000000000.timeindex                              │ │
│  │  Size: 1 GB, Offsets: 0 - 999,999                        │ │
│  │                                                           │ │
│  │  00000000001000000.log (segment 2)                        │ │
│  │  00000000001000000.index                                  │ │
│  │  00000000001000000.timeindex                              │ │
│  │  Size: 1 GB, Offsets: 1,000,000 - 1,229,999              │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                  IMPORTANT OFFSETS                        │ │
│  │                                                           │ │
│  │  Log Start Offset (LSO): 0                                │ │
│  │    ↓ First available message                              │ │
│  │    (Can be > 0 if old data deleted)                       │ │
│  │                                                           │ │
│  │  High Water Mark (HWM): 1,234,567                         │ │
│  │    ↓ Last offset replicated to all ISR                    │ │
│  │    (Consumers can only read up to HWM)                    │ │
│  │                                                           │ │
│  │  Log End Offset (LEO): 1,234,570                          │ │
│  │    ↓ Next offset to be written                            │ │
│  │    (Newer messages not yet replicated)                    │ │
│  │                                                           │ │
│  │  Last Stable Offset (LSO): 1,234,567                      │ │
│  │    ↓ For transactional messages                           │ │
│  │    (Last offset of committed transactions)                │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  VISUAL REPRESENTATION:                                         │
│                                                                 │
│  Offset:  0 ──────────────────────────────────▶ 1,234,570     │
│           │                                       │             │
│           │◄──── Consumers can read ────────────▶│             │
│           │                                       │             │
│           LSO                                    HWM  LEO       │
│        (start)                              (replicated) (end)  │
│                                                                 │
│  Messages between HWM and LEO:                                  │
│  - Written to leader                                            │
│  - Not yet replicated to all ISR                                │
│  - Not visible to consumers (for consistency)                   │
└─────────────────────────────────────────────────────────────────┘
```

### Offset Types Explained:

**Log Start Offset (LSO):**
```
What: First available offset in partition
Why: Old messages get deleted (retention)
Example:
  - Initially: LSO = 0
  - After 7 days: Messages 0-1000 deleted, LSO = 1001
```

**High Water Mark (HWM):**
```
What: Highest offset replicated to ALL ISR members
Why: Ensures consistency - consumers only see replicated data
Example:
  - Leader has offsets 0-100
  - Follower 1 has 0-100
  - Follower 2 has 0-98
  - HWM = 98 (lowest common offset)
```

**Log End Offset (LEO):**
```
What: Offset of next message to be appended
Why: Tracks end of log on each replica
Example:
  - Leader LEO: 100 (has 0-99)
  - Follower LEO: 98 (has 0-97)
  - Follower is 2 messages behind
```

**Last Stable Offset (LSO - for transactions):**
```
What: Offset of last committed transaction
Why: Consumers reading committed data only see up to LSO
Example:
  - Offset 90-95: Transaction 1 (committed)
  - Offset 96-100: Transaction 2 (in progress)
  - LSO = 95 (consumers can read up to 95)
```

---

## Level 6: Segment Files & Storage

### Inside a Segment

```
DIRECTORY: /var/kafka/data/orders-0/

┌─────────────────────────────────────────────────────────────────┐
│                  SEGMENT FILES                                  │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐│
│  │  00000000001230000.log                                     ││
│  │  ┌──────────────────────────────────────────────────────┐ ││
│  │  │ ACTUAL MESSAGES (BINARY FORMAT)                      │ ││
│  │  │                                                      │ ││
│  │  │ Offset: 1230000                                      │ ││
│  │  │ ┌──────────────────────────────────────────────────┐│ ││
│  │  │ │ Length: 1234 bytes                               ││ ││
│  │  │ │ CRC: 0x12345678                                  ││ ││
│  │  │ │ Magic: 2 (message format version)                ││ ││
│  │  │ │ Attributes: 0 (no compression)                   ││ ││
│  │  │ │ Timestamp: 1699564800000                         ││ ││
│  │  │ │ Key Length: 12                                   ││ ││
│  │  │ │ Key: "customer-123"                              ││ ││
│  │  │ │ Value Length: 1100                               ││ ││
│  │  │ │ Value: {...order data...}                        ││ ││
│  │  │ │ Headers: [...]                                   ││ ││
│  │  │ └──────────────────────────────────────────────────┘│ ││
│  │  │                                                      │ ││
│  │  │ Offset: 1230001                                      │ ││
│  │  │ ┌──────────────────────────────────────────────────┐│ ││
│  │  │ │ [Next message...]                                ││ ││
│  │  │ └──────────────────────────────────────────────────┘│ ││
│  │  │                                                      │ ││
│  │  │ ... more messages ...                                │ ││
│  │  └──────────────────────────────────────────────────────┘ ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐│
│  │  00000000001230000.index                                   ││
│  │  ┌──────────────────────────────────────────────────────┐ ││
│  │  │ OFFSET → PHYSICAL POSITION MAPPING                   │ ││
│  │  │                                                      │ ││
│  │  │ Relative Offset | Physical Position (bytes)         │ ││
│  │  │ ────────────────┼─────────────────────              │ ││
│  │  │       0         │        0                           │ ││
│  │  │     4096        │   524288  (512 KB)                 │ ││
│  │  │     8192        │  1048576  (1 MB)                   │ ││
│  │  │    12288        │  1572864  (1.5 MB)                 │ ││
│  │  │      ...        │      ...                           │ ││
│  │  │                                                      │ ││
│  │  │ Note: Relative offset = offset - base_offset        │ ││
│  │  │       (1230000 is base, so offset 1230000 = rel 0)  │ ││
│  │  └──────────────────────────────────────────────────────┘ ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐│
│  │  00000000001230000.timeindex                               ││
│  │  ┌──────────────────────────────────────────────────────┐ ││
│  │  │ TIMESTAMP → OFFSET MAPPING                           │ ││
│  │  │                                                      │ ││
│  │  │ Timestamp           | Offset                         │ ││
│  │  │ ────────────────────┼────────                        │ ││
│  │  │ 1699564800000       │ 1230000                        │ ││
│  │  │ 1699565100000       │ 1231000                        │ ││
│  │  │ 1699565400000       │ 1232000                        │ ││
│  │  │ 1699565700000       │ 1233000                        │ ││
│  │  │      ...            │   ...                          │ ││
│  │  └──────────────────────────────────────────────────────┘ ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐│
│  │  00000000001230000.snapshot (for transactional topics)     ││
│  │  - Producer state                                          ││
│  │  - Transaction markers                                     ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐│
│  │  leader-epoch-checkpoint                                   ││
│  │  - Tracks leader epochs                                    ││
│  │  - Used for log reconciliation after leader changes        ││
│  └────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### Message Format (Wire Protocol)

```
┌─────────────────────────────────────────────────────────────────┐
│                    KAFKA MESSAGE FORMAT v2                      │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ RECORD BATCH (Multiple records compressed together)      │ │
│  │                                                           │ │
│  │  Base Offset: 8 bytes                                     │ │
│  │  Batch Length: 4 bytes                                    │ │
│  │  Partition Leader Epoch: 4 bytes                          │ │
│  │  Magic: 1 byte (value = 2)                                │ │
│  │  CRC: 4 bytes (checksum)                                  │ │
│  │  Attributes: 2 bytes                                      │ │
│  │    - Compression (bits 0-2): none/gzip/snappy/lz4/zstd   │ │
│  │    - Timestamp type (bit 3): create time / log append    │ │
│  │    - Transactional (bit 4): 0 = no, 1 = yes              │ │
│  │    - Control batch (bit 5): 0 = data, 1 = control        │ │
│  │  Last Offset Delta: 4 bytes                               │ │
│  │  First Timestamp: 8 bytes                                 │ │
│  │  Max Timestamp: 8 bytes                                   │ │
│  │  Producer ID: 8 bytes                                     │ │
│  │  Producer Epoch: 2 bytes                                  │ │
│  │  Base Sequence: 4 bytes                                   │ │
│  │  Records Count: 4 bytes                                   │ │
│  │                                                           │ │
│  │  ┌─────────────────────────────────────────────────────┐ │ │
│  │  │ RECORD 1                                            │ │ │
│  │  │  Length: varint                                     │ │ │
│  │  │  Attributes: 1 byte (unused)                        │ │ │
│  │  │  Timestamp Delta: varint                            │ │ │
│  │  │  Offset Delta: varint                               │ │ │
│  │  │  Key Length: varint                                 │ │ │
│  │  │  Key: N bytes                                       │ │ │
│  │  │  Value Length: varint                               │ │ │
│  │  │  Value: N bytes                                     │ │ │
│  │  │  Headers Count: varint                              │ │ │
│  │  │    Header 1: key + value                            │ │ │
│  │  │    Header 2: key + value                            │ │ │
│  │  │    ...                                              │ │ │
│  │  └─────────────────────────────────────────────────────┘ │ │
│  │                                                           │ │
│  │  ┌─────────────────────────────────────────────────────┐ │ │
│  │  │ RECORD 2                                            │ │ │
│  │  │  [Same structure...]                                │ │ │
│  │  └─────────────────────────────────────────────────────┘ │ │
│  │                                                           │ │
│  │  ... more records ...                                     │ │
│  └───────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘

KEY CONCEPTS:

1. Record Batch = Multiple messages compressed together
   - More efficient than per-message compression
   - Reduces per-message overhead

2. Varint = Variable-length integer
   - Small numbers use fewer bytes
   - Saves space

3. Delta encoding:
   - Timestamp delta from base timestamp
   - Offset delta from base offset
   - Saves space

4. Headers:
   - Key-value pairs for metadata
   - Example: trace-id, correlation-id, schema-version
```

### File System Layout

```
/var/kafka/data/
│
├── orders-0/                    (Topic: orders, Partition: 0)
│   ├── 00000000000000000000.log
│   ├── 00000000000000000000.index
│   ├── 00000000000000000000.timeindex
│   ├── 00000000001000000000.log
│   ├── 00000000001000000000.index
│   ├── 00000000001000000000.timeindex
│   ├── 00000000002000000000.log    (Active segment)
│   ├── 00000000002000000000.index
│   ├── 00000000002000000000.timeindex
│   ├── leader-epoch-checkpoint
│   └── partition.metadata
│
├── orders-1/                    (Topic: orders, Partition: 1)
│   ├── [similar structure]
│   └── ...
│
├── orders-2/                    (Topic: orders, Partition: 2)
│   ├── [similar structure]
│   └── ...
│
├── customers-0/                 (Topic: customers, Partition: 0)
│   ├── [similar structure]
│   └── ...
│
└── __consumer_offsets-0/       (Internal topic for consumer offsets)
    ├── [similar structure]
    └── ...

NOTES:
- File names are base offset (padded to 20 digits)
- .log = actual messages
- .index = offset → byte position
- .timeindex = timestamp → offset
- Active segment = currently being written
- Old segments closed when size or time limit reached
```

---

## Level 7: Producer Architecture

### Producer Internal Components

```
┌─────────────────────────────────────────────────────────────────┐
│                    KAFKA PRODUCER                               │
│                                                                 │
│  APPLICATION CODE:                                              │
│  ┌────────────────────────────────────────────────────────────┐│
│  │ producer.send(                                             ││
│  │   new ProducerRecord<>("orders", key, value)               ││
│  │ );                                                         ││
│  └────────────────────────────────────────────────────────────┘│
│                          │                                      │
│                          ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐│
│  │                  SERIALIZERS                               ││
│  │  - Key Serializer (String → bytes)                        ││
│  │  - Value Serializer (Object → bytes)                      ││
│  └────────────────────────────────────────────────────────────┘│
│                          │                                      │
│                          ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐│
│  │                  PARTITIONER                               ││
│  │                                                            ││
│  │  If key present:                                           ││
│  │    partition = hash(key) % num_partitions                  ││
│  │                                                            ││
│  │  If no key:                                                ││
│  │    - Round-robin (old default)                             ││
│  │    - Sticky partitioner (new default, better batching)     ││
│  │                                                            ││
│  │  Custom partitioner:                                       ││
│  │    - Implement Partitioner interface                       ││
│  │    - Custom logic for partition selection                  ││
│  └────────────────────────────────────────────────────────────┘│
│                          │                                      │
│                          ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐│
│  │              RECORD ACCUMULATOR (BUFFER)                   ││
│  │                                                            ││
│  │  Per-partition queues:                                     ││
│  │                                                            ││
│  │  Partition 0:  [Batch1][Batch2][Batch3]                   ││
│  │  Partition 1:  [Batch1][Batch2]                           ││
│  │  Partition 2:  [Batch1]                                    ││
│  │                                                            ││
│  │  Each batch:                                               ││
│  │  - Max size: batch.size (default 16 KB)                    ││
│  │  - Max wait: linger.ms (default 0 ms)                      ││
│  │  - Compression: compression.type (none/gzip/snappy/lz4)    ││
│  │                                                            ││
│  │  Total buffer size: buffer.memory (default 32 MB)          ││
│  │                                                            ││
│  │  If buffer full:                                           ││
│  │  - Block for max.block.ms (default 60s)                    ││
│  │  - Then throw exception                                    ││
│  └────────────────────────────────────────────────────────────┘│
│                          │                                      │
│                          ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐│
│  │                    SENDER THREAD                           ││
│  │                   (Background I/O)                         ││
│  │                                                            ││
│  │  While (running):                                          ││
│  │    1. Get ready batches from accumulator                   ││
│  │    2. Group by broker                                      ││
│  │    3. Create ProduceRequests                               ││
│  │    4. Send to brokers                                      ││
│  │    5. Handle responses/retries                             ││
│  │                                                            ││
│  │  Max in-flight requests per connection:                    ││
│  │    max.in.flight.requests.per.connection (default 5)       ││
│  │                                                            ││
│  │  Request timeout:                                          ││
│  │    request.timeout.ms (default 30s)                        ││
│  └────────────────────────────────────────────────────────────┘│
│                          │                                      │
│                          ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐│
│  │                  NETWORK CLIENT                            ││
│  │                                                            ││
│  │  Connection pool:                                          ││
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    ││
│  │  │ Broker 1     │  │ Broker 2     │  │ Broker 3     │    ││
│  │  │ Socket       │  │ Socket       │  │ Socket       │    ││
│  │  └──────────────┘  └──────────────┘  └──────────────┘    ││
│  │                                                            ││
│  │  Manages:                                                  ││
│  │  - TCP connections                                         ││
│  │  - Request/response correlation                            ││
│  │  - Connection failures/retries                             ││
│  └────────────────────────────────────────────────────────────┘│
│                          │                                      │
│                          ▼                                      │
│                  KAFKA CLUSTER                                  │
│           (Brokers receive ProduceRequests)                     │
└─────────────────────────────────────────────────────────────────┘
```

### Producer Flow Step-by-Step

```
1. Application calls producer.send()
   ↓
2. Interceptors (optional, for monitoring/modification)
   ↓
3. Serializer converts key & value to bytes
   ↓
4. Partitioner determines target partition
   ↓
5. Message added to RecordAccumulator buffer
   ↓
6. Accumulator batches messages by partition
   ↓
7. Sender thread (when batch ready OR linger.ms expires):
   - Pulls batches from accumulator
   - Groups by destination broker
   - Creates ProduceRequest
   ↓
8. Network layer sends request to broker
   ↓
9. Broker processes request:
   - Validates
   - Appends to log
   - Replicates to followers (if acks=all)
   - Sends response
   ↓
10. Sender receives response:
    - Success: invoke callback
    - Retriable error: retry (up to retries times)
    - Non-retriable error: fail immediately
   ↓
11. Callback executed in application
```

### Producer Guarantees (acks setting)

```
acks=0 (No acknowledgment):
  Producer              Broker
     │                    │
     ├──── Message ──────▶│
     │                    │ (may or may not write)
     │ (returns immediate)│
     │                    │
  Fast, no durability

acks=1 (Leader acknowledgment):
  Producer              Leader           Follower
     │                    │                 │
     ├──── Message ──────▶│                 │
     │                    ├─ Write to log   │
     │                    ├─ Replicate ────▶│
     │◀──── Ack ──────────┤                 │ (async replication)
     │                    │                 │
  Balanced (most common)

acks=all/-1 (All ISR acknowledgment):
  Producer              Leader           Follower 1      Follower 2
     │                    │                 │               │
     ├──── Message ──────▶│                 │               │
     │                    ├─ Write to log   │               │
     │                    ├─ Replicate ────▶│               │
     │                    │                 ├─ Ack ────────▶│
     │                    ├─ Replicate ─────┼──────────────▶│
     │                    │                 │               ├─ Ack ─▶│
     │                    │◀──── Acks ──────┴───────────────┘
     │◀──── Ack ──────────┤
     │                    │
  Strongest durability (slowest)
```

---

## Level 8: Consumer Architecture

### Consumer Internal Components

```
┌─────────────────────────────────────────────────────────────────┐
│                    KAFKA CONSUMER                               │
│                                                                 │
│  APPLICATION CODE:                                              │
│  ┌────────────────────────────────────────────────────────────┐│
│  │ while (true) {                                             ││
│  │   records = consumer.poll(Duration.ofMillis(100));         ││
│  │   for (record : records) {                                 ││
│  │     process(record);                                       ││
│  │   }                                                        ││
│  │ }                                                          ││
│  └────────────────────────────────────────────────────────────┘│
│                          │                                      │
│                          ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐│
│  │              CONSUMER COORDINATOR                          ││
│  │                                                            ││
│  │  Responsibilities:                                         ││
│  │  - Join consumer group                                     ││
│  │  - Participate in rebalance                                ││
│  │  - Maintain membership (heartbeats)                        ││
│  │  - Commit offsets                                          ││
│  │                                                            ││
│  │  Group state:                                              ││
│  │  - group.id: "order-processors"                            ││
│  │  - member.id: "consumer-1-uuid"                            ││
│  │  - generation.id: 5                                        ││
│  │                                                            ││
│  │  Heartbeat:                                                ││
│  │  - Sent every heartbeat.interval.ms (default 3s)           ││
│  │  - If missed for session.timeout.ms (default 10s):         ││
│  │    → Consumer considered dead                              ││
│  │    → Rebalance triggered                                   ││
│  └────────────────────────────────────────────────────────────┘│
│                          │                                      │
│                          ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐│
│  │                FETCHER / FETCH MANAGER                     ││
│  │                                                            ││
│  │  Assigned partitions:                                      ││
│  │  - orders-0                                                ││
│  │  - orders-2                                                ││
│  │                                                            ││
│  │  Fetch positions:                                          ││
│  │  - orders-0: offset 12345                                  ││
│  │  - orders-2: offset 67890                                  ││
│  │                                                            ││
│  │  Fetching parameters:                                      ││
│  │  - fetch.min.bytes (default 1): Min data before return     ││
│  │  - fetch.max.wait.ms (default 500ms): Max wait time        ││
│  │  - max.partition.fetch.bytes (default 1 MB): Per partition ││
│  │  - max.poll.records (default 500): Max records per poll    ││
│  │                                                            ││
│  │  Prefetching:                                              ││
│  │  - Fetches in background while app processes              ││
│  │  - Maintains local buffer                                  ││
│  └────────────────────────────────────────────────────────────┘│
│                          │                                      │
│                          ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐│
│  │                  NETWORK CLIENT                            ││
│  │                                                            ││
│  │  Connections to brokers:                                   ││
│  │  - One connection per broker                               ││
│  │  - Sends FetchRequests                                     ││
│  │  - Sends HeartbeatRequests                                 ││
│  │  - Sends OffsetCommitRequests                              ││
│  └────────────────────────────────────────────────────────────┘│
│                          │                                      │
│                          ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐│
│  │                 DESERIALIZERS                              ││
│  │  - Key Deserializer (bytes → String)                      ││
│  │  - Value Deserializer (bytes → Object)                    ││
│  └────────────────────────────────────────────────────────────┘│
│                          │                                      │
│                          ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐│
│  │              OFFSET MANAGEMENT                             ││
│  │                                                            ││
│  │  Current offsets (in memory):                              ││
│  │  - orders-0: 12345                                         ││
│  │  - orders-2: 67890                                         ││
│  │                                                            ││
│  │  Committed offsets (in __consumer_offsets):                ││
│  │  - orders-0: 12300                                         ││
│  │  - orders-2: 67850                                         ││
│  │                                                            ││
│  │  Commit strategy:                                          ││
│  │  - Auto commit: enable.auto.commit=true                    ││
│  │    → Commits every auto.commit.interval.ms (5s)            ││
│  │  - Manual commit: consumer.commitSync() / commitAsync()    ││
│  └────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### Consumer Group Coordination

```
┌─────────────────────────────────────────────────────────────────┐
│                      CONSUMER GROUP                             │
│                  Group ID: "order-processors"                   │
│                                                                 │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐   │
│  │  Consumer 1    │  │  Consumer 2    │  │  Consumer 3    │   │
│  │  member-1-uuid │  │  member-2-uuid │  │  member-3-uuid │   │
│  └────────────────┘  └────────────────┘  └────────────────┘   │
│          │                   │                   │             │
│          │                   │                   │             │
│          └───────────────────┼───────────────────┘             │
│                              │                                 │
│                              ▼                                 │
│              ┌───────────────────────────────┐                 │
│              │   GROUP COORDINATOR           │                 │
│              │   (Broker elected as coord)   │                 │
│              │                               │                 │
│              │  Manages:                     │                 │
│              │  - Group membership           │                 │
│              │  - Partition assignments      │                 │
│              │  - Offset commits             │                 │
│              │  - Rebalance protocol         │                 │
│              └───────────────────────────────┘                 │
│                              │                                 │
│                              ▼                                 │
│              ┌───────────────────────────────┐                 │
│              │  PARTITION ASSIGNMENT         │                 │
│              │                               │                 │
│              │  Topic: orders (3 partitions) │                 │
│              │                               │                 │
│              │  Consumer 1 → [Partition 0]   │                 │
│              │  Consumer 2 → [Partition 1]   │                 │
│              │  Consumer 3 → [Partition 2]   │                 │
│              └───────────────────────────────┘                 │
└─────────────────────────────────────────────────────────────────┘

REBALANCE TRIGGERS:
1. Consumer joins group (new consumer)
2. Consumer leaves group (shutdown/crash)
3. Consumer heartbeat timeout (appears dead)
4. Topic partition count changes
5. Consumer calls unsubscribe()

REBALANCE PROTOCOL:

Phase 1: Join Group
  Consumers ──────▶ JoinGroupRequest ──────▶ Coordinator
                                                  │
  All members send JoinGroupRequest               │
                                                  │
  Coordinator ◀─── JoinGroupResponse ◀────────────┤
                   (Assigns one as leader)

Phase 2: Sync Group
  Leader calculates partition assignment
  │
  Leader ──────▶ SyncGroupRequest ──────▶ Coordinator
                 (with assignments)            │
                                               │
  All members ◀─── SyncGroupResponse ◀─────────┤
                   (get their assignments)

Phase 3: Heartbeat
  Consumers send heartbeats to maintain membership
  If heartbeat missed → Rebalance
```

### Consumer Offset Management

```
┌─────────────────────────────────────────────────────────────────┐
│              OFFSET STORAGE (__consumer_offsets topic)          │
│                                                                 │
│  This is a special internal Kafka topic                         │
│  - Stores consumer group offsets                                │
│  - Compacted topic (keeps latest offset per partition)          │
│  - 50 partitions by default                                     │
│  - Replication factor typically 3                               │
│                                                                 │
│  Key format:                                                    │
│  [group.id, topic, partition] → offset                          │
│                                                                 │
│  Example entries:                                               │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Key                                      │ Value          │ │
│  ├──────────────────────────────────────────┼────────────────┤ │
│  │ [order-processors, orders, 0]            │ 12345          │ │
│  │ [order-processors, orders, 1]            │ 67890          │ │
│  │ [order-processors, orders, 2]            │ 99999          │ │
│  │ [payment-processors, payments, 0]        │ 55555          │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  OFFSET COMMIT STRATEGIES:                                      │
│                                                                 │
│  1. Auto-commit (enable.auto.commit=true):                      │
│     ┌──────────────────────────────────────────────────────┐   │
│     │ consumer.poll()  →  Process  →  [Auto commit after]  │   │
│     │                                   5 seconds           │   │
│     └──────────────────────────────────────────────────────┘   │
│     Risk: May commit offsets for unprocessed messages          │
│                                                                 │
│  2. Manual sync commit:                                         │
│     ┌──────────────────────────────────────────────────────┐   │
│     │ records = consumer.poll()                            │   │
│     │ for (record : records) {                             │   │
│     │   process(record);                                   │   │
│     │ }                                                    │   │
│     │ consumer.commitSync();  ← Blocks until ack           │   │
│     └──────────────────────────────────────────────────────┘   │
│     Safe but slower                                            │
│                                                                 │
│  3. Manual async commit:                                        │
│     ┌──────────────────────────────────────────────────────┐   │
│     │ records = consumer.poll()                            │   │
│     │ for (record : records) {                             │   │
│     │   process(record);                                   │   │
│     │ }                                                    │   │
│     │ consumer.commitAsync();  ← Non-blocking              │   │
│     └──────────────────────────────────────────────────────┘   │
│     Fast but may lose commits on failure                       │
│                                                                 │
│  4. Manual commit per record (expensive):                       │
│     ┌──────────────────────────────────────────────────────┐   │
│     │ for (record : records) {                             │   │
│     │   process(record);                                   │   │
│     │   consumer.commitSync(record.offset);                │   │
│     │ }                                                    │   │
│     └──────────────────────────────────────────────────────┘   │
│     Slowest but most precise                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Level 9: Request Flow & Protocols

### Request/Response Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    PRODUCE REQUEST FLOW                         │
│                                                                 │
│  PRODUCER                    BROKER (LEADER)         FOLLOWERS  │
│     │                             │                      │      │
│     │                             │                      │      │
│  1. │── ProduceRequest ──────────▶│                      │      │
│     │   (topic, partition,        │                      │      │
│     │    messages, acks)           │                      │      │
│     │                             │                      │      │
│     │                          2. │─ Validate request    │      │
│     │                             │  - Auth check        │      │
│     │                             │  - Partition exists  │      │
│     │                             │  - Am I leader?      │      │
│     │                             │                      │      │
│     │                          3. │─ Append to log       │      │
│     │                             │  (write to disk)     │      │
│     │                             │                      │      │
│     │                          4. │─ Update LEO          │      │
│     │                             │                      │      │
│     │                             │                      │      │
│     │      [If acks=1]         5. │                      │      │
│     │◀── ProduceResponse ─────────┤                      │      │
│     │   (success)                 │                      │      │
│     │                             │                      │      │
│     │                             │                      │      │
│     │      [If acks=all]          │                      │      │
│     │                          6. │─ Replicate ─────────▶│      │
│     │                             │                      │      │
│     │                             │                   7. │─ Fetch│
│     │                             │                      │  data │
│     │                             │                      │      │
│     │                             │                   8. │─ Write│
│     │                             │                      │  log  │
│     │                             │                      │      │
│     │                             │◀─ FetchResponse ──── 9.     │
│     │                             │   (ack)              │      │
│     │                             │                      │      │
│     │                         10. │─ Update HWM          │      │
│     │                             │  (all replicas       │      │
│     │                             │   caught up)         │      │
│     │                             │                      │      │
│     │◀── ProduceResponse ───── 11.                      │      │
│     │   (success)                 │                      │      │
│     │                             │                      │      │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                     FETCH REQUEST FLOW                          │
│                                                                 │
│  CONSUMER                    BROKER (LEADER)                    │
│     │                             │                             │
│     │                             │                             │
│  1. │── FetchRequest ────────────▶│                             │
│     │   (topic, partition,        │                             │
│     │    offset, max_bytes)       │                             │
│     │                             │                             │
│     │                          2. │─ Validate request           │
│     │                             │  - Auth check               │
│     │                             │  - Partition exists         │
│     │                             │  - Offset valid             │
│     │                             │                             │
│     │                          3. │─ Check data available       │
│     │                             │  at requested offset        │
│     │                             │                             │
│     │                             │                             │
│     │   [If data available]       │                             │
│     │                          4. │─ Read from disk/cache       │
│     │                             │  (up to HWM)                │
│     │                             │                             │
│     │◀── FetchResponse ────────5. │                             │
│     │   (records, metadata)       │                             │
│     │                             │                             │
│     │                             │                             │
│     │   [If no data yet]          │                             │
│     │                          6. │─ Wait for                   │
│     │                             │  fetch.max.wait.ms          │
│     │                             │  OR                         │
│     │                             │  fetch.min.bytes            │
│     │                             │  available                  │
│     │                             │                             │
│     │◀── FetchResponse ────────7. │                             │
│     │   (may be empty)            │                             │
│     │                             │                             │
└─────────────────────────────────────────────────────────────────┘
```

### API Request Types

```
┌─────────────────────────────────────────────────────────────────┐
│                    KAFKA API REQUESTS                           │
│                                                                 │
│  PRODUCER APIs:                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ ProduceRequest                                           │  │
│  │   - Write messages to partition                          │  │
│  │   - Specify acks level                                   │  │
│  │   - Transactional support                                │  │
│  │                                                          │  │
│  │ InitProducerIdRequest (for transactions)                 │  │
│  │   - Get producer ID and epoch                            │  │
│  │                                                          │  │
│  │ AddPartitionsToTxnRequest (for transactions)             │  │
│  │   - Add partitions to current transaction                │  │
│  │                                                          │  │
│  │ EndTxnRequest (for transactions)                         │  │
│  │   - Commit or abort transaction                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  CONSUMER APIs:                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ FetchRequest                                             │  │
│  │   - Read messages from partition                         │  │
│  │   - Can fetch from multiple partitions                   │  │
│  │                                                          │  │
│  │ ListOffsetsRequest                                       │  │
│  │   - Get earliest/latest offset                           │  │
│  │   - Get offset by timestamp                              │  │
│  │                                                          │  │
│  │ OffsetFetchRequest                                       │  │
│  │   - Get committed offset for consumer group              │  │
│  │                                                          │  │
│  │ OffsetCommitRequest                                      │  │
│  │   - Commit offset for consumer group                     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  CONSUMER GROUP APIs:                                           │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ JoinGroupRequest                                         │  │
│  │   - Join consumer group                                  │  │
│  │   - Triggers rebalance                                   │  │
│  │                                                          │  │
│  │ SyncGroupRequest                                         │  │
│  │   - Get partition assignment after rebalance             │  │
│  │                                                          │  │
│  │ HeartbeatRequest                                         │  │
│  │   - Keep consumer group membership alive                 │  │
│  │                                                          │  │
│  │ LeaveGroupRequest                                        │  │
│  │   - Voluntarily leave consumer group                     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  METADATA APIs:                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ MetadataRequest                                          │  │
│  │   - Get topic/partition/broker info                      │  │
│  │   - Find leader for partition                            │  │
│  │                                                          │  │
│  │ ApiVersionsRequest                                       │  │
│  │   - Get supported API versions                           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ADMIN APIs:                                                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ CreateTopicsRequest                                      │  │
│  │ DeleteTopicsRequest                                      │  │
│  │ CreatePartitionsRequest                                  │  │
│  │ DeleteRecordsRequest                                     │  │
│  │ AlterConfigsRequest                                      │  │
│  │ DescribeConfigsRequest                                   │  │
│  │ DescribeGroupsRequest                                    │  │
│  │ ListGroupsRequest                                        │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Level 10: Replication & High Availability

### Replication Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                PARTITION REPLICATION                            │
│            Topic: orders, Partition: 0                          │
│            Replication Factor: 3                                │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐│
│  │                      BROKER 1 (LEADER)                     ││
│  │                                                            ││
│  │  Partition: orders-0                                       ││
│  │  Role: Leader                                              ││
│  │                                                            ││
│  │  Offsets: 0 ─────────────────────────────────▶ 1000       ││
│  │           [============= MESSAGES ==============]          ││
│  │                                                            ││
│  │  LEO (Log End Offset): 1000                                ││
│  │  HWM (High Water Mark): 998                                ││
│  │                                                            ││
│  │  Handles:                                                  ││
│  │  - All produce requests                                    ││
│  │  - All fetch requests (from consumers)                     ││
│  │  - Coordinates replication                                 ││
│  │  - Updates HWM                                             ││
│  └────────────────────────────────────────────────────────────┘│
│                          │     │                                │
│                          │     │                                │
│            Replication   │     │   Replication                  │
│                    ┌─────┘     └─────┐                          │
│                    │                 │                          │
│                    ▼                 ▼                          │
│  ┌─────────────────────────────┐  ┌────────────────────────┐  │
│  │    BROKER 2 (FOLLOWER)      │  │   BROKER 3 (FOLLOWER)  │  │
│  │                             │  │                        │  │
│  │  Partition: orders-0        │  │  Partition: orders-0   │  │
│  │  Role: Follower (ISR)       │  │  Role: Follower (ISR)  │  │
│  │                             │  │                        │  │
│  │  Offsets: 0 ──────▶ 998     │  │  Offsets: 0 ───▶ 998   │  │
│  │  [====== MESSAGES =======]  │  │  [==== MESSAGES ====]  │  │
│  │                             │  │                        │  │
│  │  LEO: 998                   │  │  LEO: 998              │  │
│  │                             │  │                        │  │
│  │  Continuously fetches from  │  │  Continuously fetches  │  │
│  │  leader (FetchRequest)      │  │  from leader           │  │
│  │                             │  │                        │  │
│  │  In ISR: YES                │  │  In ISR: YES           │  │
│  └─────────────────────────────┘  └────────────────────────┘  │
│                                                                 │
│  ISR (In-Sync Replicas): [Broker 1, Broker 2, Broker 3]        │
│                                                                 │
│  Committed messages (HWM = 998):                                │
│    - Replicated to all ISR members                              │
│    - Safe to consume                                            │
│                                                                 │
│  Uncommitted messages (998-1000):                               │
│    - Only on leader                                             │
│    - Not yet replicated                                         │
│    - Not visible to consumers                                   │
└─────────────────────────────────────────────────────────────────┘
```

### ISR Management

```
┌─────────────────────────────────────────────────────────────────┐
│               IN-SYNC REPLICA (ISR) MANAGEMENT                  │
│                                                                 │
│  A follower is in ISR if:                                       │
│  1. It has sent a fetch request in last replica.lag.time.max.ms│
│     (default 10 seconds)                                        │
│  2. It has caught up to leader's log end offset                 │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐│
│  │                    NORMAL OPERATION                        ││
│  │                                                            ││
│  │  Leader (Broker 1):                                        ││
│  │    LEO: 1000                                               ││
│  │    HWM: 998                                                ││
│  │                                                            ││
│  │  Follower 1 (Broker 2):                                    ││
│  │    LEO: 998                                                ││
│  │    Last fetch: 1 second ago  ✓ IN ISR                     ││
│  │                                                            ││
│  │  Follower 2 (Broker 3):                                    ││
│  │    LEO: 998                                                ││
│  │    Last fetch: 2 seconds ago  ✓ IN ISR                    ││
│  │                                                            ││
│  │  ISR: [1, 2, 3]                                            ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐│
│  │              FOLLOWER FALLING BEHIND                       ││
│  │                                                            ││
│  │  Leader (Broker 1):                                        ││
│  │    LEO: 1500                                               ││
│  │    HWM: 1498                                               ││
│  │                                                            ││
│  │  Follower 1 (Broker 2):                                    ││
│  │    LEO: 1498                                               ││
│  │    Last fetch: 2 seconds ago  ✓ IN ISR                    ││
│  │                                                            ││
│  │  Follower 2 (Broker 3):  [NETWORK ISSUES]                 ││
│  │    LEO: 1200                                               ││
│  │    Last fetch: 15 seconds ago  ✗ REMOVED FROM ISR         ││
│  │                                                            ││
│  │  Old ISR: [1, 2, 3]                                        ││
│  │  New ISR: [1, 2]          ← Controller updates            ││
│  │                              ZooKeeper/metadata            ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐│
│  │              FOLLOWER CATCHING UP                          ││
│  │                                                            ││
│  │  Follower 2 (Broker 3):  [NETWORK RESTORED]               ││
│  │    LEO: 1200 → 1300 → 1400 → 1498                         ││
│  │    Catching up...                                          ││
│  │                                                            ││
│  │  Once caught up:                                           ││
│  │    LEO: 1498                                               ││
│  │    Last fetch: 1 second ago  ✓ ADDED BACK TO ISR          ││
│  │                                                            ││
│  │  ISR: [1, 2, 3]  ← Restored                               ││
│  └────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### Leader Election

```
┌─────────────────────────────────────────────────────────────────┐
│                      LEADER ELECTION                            │
│                                                                 │
│  SCENARIO: Leader fails                                         │
│                                                                 │
│  BEFORE FAILURE:                                                │
│  ┌────────────────────────────────────────────────────────────┐│
│  │  Broker 1 (Leader) ──▶ LEO: 1000, ISR: [1,2,3]           ││
│  │  Broker 2 (Follower) ─▶ LEO: 998                          ││
│  │  Broker 3 (Follower) ─▶ LEO: 998                          ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
│  FAILURE:                                                       │
│  ┌────────────────────────────────────────────────────────────┐│
│  │  Broker 1: ✗✗✗ CRASHED ✗✗✗                                ││
│  └────────────────────────────────────────────────────────────┘│
│                          │                                      │
│                          ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐│
│  │  Controller detects failure (heartbeat timeout)            ││
│  │  - Broker 1 didn't send heartbeat for 6 seconds            ││
│  │  - Controller marks Broker 1 as down                       ││
│  └────────────────────────────────────────────────────────────┘│
│                          │                                      │
│                          ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐│
│  │  Controller initiates leader election:                     ││
│  │                                                            ││
│  │  1. Get current ISR: [1, 2, 3]                             ││
│  │  2. Remove dead broker: [2, 3]                             ││
│  │  3. Select new leader from ISR (first available):          ││
│  │     → Broker 2 (preferred replica order)                   ││
│  │                                                            ││
│  │  4. Update metadata:                                       ││
│  │     - New leader: Broker 2                                 ││
│  │     - New ISR: [2, 3]                                      ││
│  │     - Increment leader epoch                               ││
│  │                                                            ││
│  │  5. Send LeaderAndIsrRequest to:                           ││
│  │     - Broker 2 (you are now leader)                        ││
│  │     - Broker 3 (new leader is Broker 2)                    ││
│  │                                                            ││
│  │  6. Send UpdateMetadata to all brokers                     ││
│  └────────────────────────────────────────────────────────────┘│
│                          │                                      │
│                          ▼                                      │
│  AFTER ELECTION:                                                │
│  ┌────────────────────────────────────────────────────────────┐│
│  │  Broker 2 (NEW LEADER) ──▶ LEO: 998, ISR: [2,3]          ││
│  │  Broker 3 (Follower) ─────▶ LEO: 998                      ││
│  │                                                            ││
│  │  - Producers redirect to Broker 2                          ││
│  │  - Consumers redirect to Broker 2                          ││
│  │  - Broker 3 continues replicating from Broker 2            ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
│  UNCLEAN LEADER ELECTION:                                       │
│  ┌────────────────────────────────────────────────────────────┐│
│  │  What if ALL ISR members are down?                         ││
│  │                                                            ││
│  │  unclean.leader.election.enable = false (default):         ││
│  │    - Wait for ISR member to come back                      ││
│  │    - No data loss, but unavailable                         ││
│  │    - Preferred for critical data                           ││
│  │                                                            ││
│  │  unclean.leader.election.enable = true:                    ││
│  │    - Elect non-ISR replica as leader                       ││
│  │    - Potential data loss (may be behind)                   ││
│  │    - Availability over consistency                         ││
│  │    - Use for non-critical data                             ││
│  └────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### Handling Failures

```
┌─────────────────────────────────────────────────────────────────┐
│                    FAILURE SCENARIOS                            │
│                                                                 │
│  1. PRODUCER FAILURE:                                           │
│  ┌────────────────────────────────────────────────────────────┐│
│  │  Producer crashes after sending message                    ││
│  │                                                            ││
│  │  Without transactions:                                     ││
│  │    - Message may be duplicated (if retry after success)    ││
│  │    - At-least-once delivery                                ││
│  │                                                            ││
│  │  With transactions (enable.idempotence=true):              ││
│  │    - Producer gets unique ID                               ││
│  │    - Messages get sequence numbers                         ││
│  │    - Broker deduplicates                                   ││
│  │    - Exactly-once delivery                                 ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
│  2. CONSUMER FAILURE:                                           │
│  ┌────────────────────────────────────────────────────────────┐│
│  │  Consumer crashes during processing                        ││
│  │                                                            ││
│  │  If offsets auto-committed:                                ││
│  │    - May have committed before processing                  ││
│  │    - Messages lost (at-most-once)                          ││
│  │                                                            ││
│  │  If manual commit after processing:                        ││
│  │    - Offset not committed                                  ││
│  │    - Messages reprocessed (at-least-once)                  ││
│  │                                                            ││
│  │  Heartbeat timeout:                                        ││
│  │    - Consumer removed from group                           ││
│  │    - Rebalance triggered                                   ││
│  │    - Partitions reassigned                                 ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
│  3. BROKER FAILURE:                                             │
│  ┌────────────────────────────────────────────────────────────┐│
│  │  Follower fails:                                           ││
│  │    - Removed from ISR after replica.lag.time.max.ms        ││
│  │    - No impact on availability                             ││
│  │    - Can rejoin ISR when caught up                         ││
│  │                                                            ││
│  │  Leader fails:                                             ││
│  │    - Controller detects failure                            ││
│  │    - Elects new leader from ISR                            ││
│  │    - Updates metadata                                      ││
│  │    - Clients redirect to new leader                        ││
│  │    - Brief unavailability during election (~milliseconds)  ││
│  │                                                            ││
│  │  Multiple brokers fail:                                    ││
│  │    - If ISR members still available: elect new leader      ││
│  │    - If all ISR down: wait OR unclean election             ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
│  4. CONTROLLER FAILURE:                                         │
│  ┌────────────────────────────────────────────────────────────┐│
│  │  Controller broker crashes                                 ││
│  │                                                            ││
│  │  Election process:                                         ││
│  │    - Remaining brokers detect failure                      ││
│  │    - New controller elected (via ZK or KRaft)              ││
│  │    - New controller loads metadata                         ││
│  │    - Resumes management duties                             ││
│  │                                                            ││
│  │  Impact:                                                   ││
│  │    - Brief delay in admin operations                       ││
│  │    - No impact on data path (produce/consume)              ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
│  5. NETWORK PARTITION:                                          │
│  ┌────────────────────────────────────────────────────────────┐│
│  │  Broker isolated from cluster                              ││
│  │                                                            ││
│  │  If follower isolated:                                     ││
│  │    - Removed from ISR                                      ││
│  │    - Can rejoin when network restored                      ││
│  │                                                            ││
│  │  If leader isolated:                                       ││
│  │    - Can't reach other brokers                             ││
│  │    - Controller elects new leader                          ││
│  │    - Old leader steps down (sees new leader epoch)         ││
│  │    - Becomes follower and truncates divergent messages     ││
│  └────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

---

## Common Conversation Topics

### Terms You'll Hear Often

**1. Throughput vs Latency:**
```
Throughput: Messages per second
  - Increased by: batching, compression, more partitions
  - Decreased by: acks=all, small batches

Latency: Time from produce to consume
  - Increased by: acks=all, linger.ms, network distance
  - Decreased by: acks=1, linger.ms=0, fewer partitions
```

**2. Exactly-once Semantics (EOS):**
```
Requires:
  - enable.idempotence=true (producer)
  - Transactions (for multi-partition atomicity)
  - isolation.level=read_committed (consumer)

How it works:
  - Producer gets unique ID
  - Messages get sequence numbers
  - Broker deduplicates
  - Transactional markers in log
```

**3. Rebalancing:**
```
When: Consumer joins/leaves, partition count changes
Impact: Brief processing pause ("stop-the-world")
Mitigation:
  - Increase session.timeout.ms
  - Decrease max.poll.interval.ms
  - Use incremental cooperative rebalancing (new)
```

**4. Compaction:**
```
Purpose: Keep latest value per key (vs time-based retention)
Use case: Database CDC, state stores
How: Background cleaner threads scan and deduplicate
Tombstones: null values signal deletion
```

**5. Consumer Lag:**
```
Definition: How far behind consumer is from latest offset
Formula: (Latest Offset) - (Consumer Offset)
Causes: Slow processing, insufficient consumers, rebalancing
Monitoring: Essential for operational health
```

**6. Partition Count:**
```
Too few:
  - Limited parallelism
  - Single consumer bottleneck

Too many:
  - More overhead (file handles, memory)
  - Longer leader election
  - More end-to-end latency

Rule of thumb: Start with (target throughput) / (consumer throughput)
```

**7. Replication Factor:**
```
RF=1: No redundancy, fast, risky
RF=2: One backup, some safety
RF=3: Two backups, standard for production
RF>3: Diminishing returns, more storage/network

Trade-off: Durability vs Storage vs Performance
```

**8. Min In-Sync Replicas:**
```
min.insync.replicas=2 with RF=3:
  - Must have 2 replicas (leader + 1 follower)
  - If only 1 replica up: refuse writes
  - Prevents data loss even if leader fails

Common pattern:
  RF=3, min.insync.replicas=2, acks=all
  → Can tolerate 1 broker failure
```

**9. Zero-copy:**
```
sendfile() system call:
  - Data sent from disk → NIC without CPU
  - Bypasses user space
  - Huge performance gain for consumers
  - Works because Kafka stores in binary format
```

**10. Log Segments:**
```
Active segment: Currently written
Closed segments: Immutable, can be deleted/compacted

Controlled by:
  - segment.bytes (default 1 GB)
  - segment.ms (default 7 days)
  - Whichever comes first
```

### Architecture Discussion Topics

**Scalability:**
```
Horizontal scaling:
  - Add more brokers → spread partitions
  - Add more partitions → more parallelism
  - Add more consumers → process faster

Limits:
  - ZooKeeper scale limit (~100-200 brokers)
  - KRaft removes this limit
  - Partition count per broker (~4000)
  - Consumer group size
```

**Consistency Model:**
```
Not strict consistency:
  - High Water Mark mechanism
  - Replication lag tolerated

Guarantees:
  - Ordered within partition
  - At-least-once by default
  - Exactly-once with transactions

Trade-offs:
  - Consistency vs Availability
  - Tunable via acks, min.insync.replicas
```

**Use Cases Discussion:**
```
Good for:
  - Event streaming (user actions, logs)
  - Messaging (queue replacement)
  - Database CDC (change data capture)
  - Metrics/monitoring aggregation
  - Log aggregation
  - Stream processing (with Kafka Streams)

Not ideal for:
  - Request/response RPC
  - Large files/blobs (MB+)
  - Sub-millisecond latency requirements
  - Strict ordering across partitions
```

**Performance Tuning:**
```
Producer:
  - batch.size, linger.ms (batching)
  - compression.type (network)
  - buffer.memory (buffering)
  - acks (durability vs speed)

Consumer:
  - fetch.min.bytes, fetch.max.wait.ms (batching)
  - max.poll.records (batch size)
  - enable.auto.commit (overhead)

Broker:
  - num.network.threads (network I/O)
  - num.io.threads (disk I/O)
  - socket.send.buffer.bytes (TCP tuning)
  - log.flush.interval.messages (disk sync)
```

### Monitoring Metrics

**Key Metrics to Monitor:**
```
Broker metrics:
  - UnderReplicatedPartitions (should be 0)
  - OfflinePartitionsCount (should be 0)
  - ActiveControllerCount (should be 1 across cluster)
  - RequestHandlerAvgIdlePercent (CPU headroom)
  - NetworkProcessorAvgIdlePercent (network headroom)
  - LogFlushRateAndTimeMs (disk performance)

Producer metrics:
  - record-send-rate
  - record-error-rate
  - request-latency-avg
  - buffer-available-bytes (memory pressure)

Consumer metrics:
  - records-lag (how far behind)
  - records-lag-max (worst partition)
  - fetch-rate
  - commit-latency-avg
  - join-rate (rebalancing frequency)
```

---

## Summary: The Complete Picture

```
┌─────────────────────────────────────────────────────────────────┐
│                    KAFKA ECOSYSTEM                              │
│                                                                 │
│                      APPLICATION LAYER                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Producers          Consumers         Stream Processing   │  │
│  │  - Java/Python      - Consumer Groups  - Kafka Streams    │  │
│  │  - Connectors       - Offset Mgmt      - ksqlDB           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              ▲│                                 │
│                              ││                                 │
│                      PROTOCOL LAYER                             │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Produce API     Fetch API      Admin API                │  │
│  │  Consumer API    Metadata API   Transaction API          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              ▲│                                 │
│                              ││                                 │
│                      KAFKA CLUSTER                              │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    BROKER NODES                           │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐              │  │
│  │  │ Broker 1 │  │ Broker 2 │  │ Broker 3 │              │  │
│  │  │          │  │          │  │          │              │  │
│  │  │ Topics   │  │ Topics   │  │ Topics   │              │  │
│  │  │ Parts    │  │ Parts    │  │ Parts    │              │  │
│  │  └──────────┘  └──────────┘  └──────────┘              │  │
│  │       │             │             │                      │  │
│  └───────┼─────────────┼─────────────┼──────────────────────┘  │
│          │             │             │                         │
│          │             │             │                         │
│                 STORAGE LAYER                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Topic → Partitions → Segments → Messages                │  │
│  │  Indexes, Offsets, Metadata                              │  │
│  │  Compaction, Retention, Replication                      │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              ▲                                  │
│                              │                                  │
│                   COORDINATION LAYER                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  ZooKeeper / KRaft                                        │  │
│  │  - Metadata                                               │  │
│  │  - Leader election                                        │  │
│  │  - Configuration                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Mental Model Summary:

1. **Topic** = Logical feed of messages (like a table)
2. **Partition** = Ordered log within topic (like a shard)
3. **Segment** = Physical file containing messages (like a chunk)
4. **Offset** = Position in partition (like primary key)
5. **Broker** = Server that stores partitions (like database node)
6. **Cluster** = Multiple brokers working together (like database cluster)
7. **Leader** = Partition's primary replica (like DB primary)
8. **Follower** = Partition's backup replica (like DB replica)
9. **ISR** = Caught-up replicas (like sync replicas)
10. **Producer** = Writes data to topics (like INSERT)
11. **Consumer** = Reads data from topics (like SELECT)
12. **Consumer Group** = Coordinated consumers (like parallel queries)
13. **Controller** = Cluster manager broker (like DB coordinator)
14. **ZooKeeper/KRaft** = Metadata store (like catalog)

---

**When someone mentions...**

- "Partition key" → Controls which partition message goes to
- "Consumer lag" → How far behind consumer is from latest
- "Rebalance" → Redistribution of partitions among consumers
- "ISR" → Replicas that are caught up with leader
- "HWM" → Last offset all replicas have (safe to read)
- "LEO" → Next offset to be written
- "Compaction" → Keeping latest value per key
- "Tombstone" → null value to signal deletion
- "acks" → How many replicas must acknowledge write
- "Batch" → Multiple messages grouped together
- "Segment" → Physical file on disk
- "Controller" → Broker managing cluster state
- "Leader epoch" → Version number for leader

You now understand the complete Kafka architecture from top to bottom!
