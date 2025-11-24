# Kafka Leader Failure: Complete Analysis
## What Happens When a Leader Broker Fails

---

## Table of Contents
1. [The Baseline Scenario](#the-baseline-scenario)
2. [Understanding Key Offsets](#understanding-key-offsets)
3. [Leader Failure Timeline](#leader-failure-timeline)
4. [Impact of Different Configurations](#impact-of-different-configurations)
5. [The Critical Question: Unreplicated Messages](#the-critical-question-unreplicated-messages)
6. [Producer Behavior During Failure](#producer-behavior-during-failure)
7. [Consumer Behavior During Failure](#consumer-behavior-during-failure)
8. [Data Loss Scenarios](#data-loss-scenarios)
9. [Configuration Best Practices](#configuration-best-practices)

---

## The Baseline Scenario

Let's establish a concrete example to work with:

```
SETUP:
Topic: orders
Partition: 0
Replication Factor: 3
Brokers: 1, 2, 3

BEFORE FAILURE:
┌─────────────────────────────────────────────────────────────────┐
│                    PARTITION: orders-0                          │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              BROKER 1 (LEADER)                           │  │
│  │                                                          │  │
│  │  Offsets: 0 ─────────────────────────────────────▶ 1005 │  │
│  │           [============ MESSAGES =============]          │  │
│  │                                                          │  │
│  │  LEO (Log End Offset): 1005                              │  │
│  │  HWM (High Water Mark): 1000                             │  │
│  │                                                          │  │
│  │  Messages 0-999:   ✓ Replicated to all followers        │  │
│  │  Messages 1000-1004: ⚠️  Only on leader (not replicated) │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              BROKER 2 (FOLLOWER)                         │  │
│  │                                                          │  │
│  │  Offsets: 0 ────────────────────────────────────▶ 1000  │  │
│  │           [============ MESSAGES =============]          │  │
│  │                                                          │  │
│  │  LEO: 1000                                               │  │
│  │  Currently fetching from leader...                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              BROKER 3 (FOLLOWER)                         │  │
│  │                                                          │  │
│  │  Offsets: 0 ────────────────────────────────────▶ 1000  │  │
│  │           [============ MESSAGES =============]          │  │
│  │                                                          │  │
│  │  LEO: 1000                                               │  │
│  │  Currently fetching from leader...                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ISR (In-Sync Replicas): [1, 2, 3]                             │
│                                                                 │
│  KEY POINT:                                                     │
│  - Messages 1000-1004 exist ONLY on Broker 1 (leader)          │
│  - These messages are NOT visible to consumers (below HWM)      │
│  - Followers are about to fetch these messages                  │
│                                                                 │
│         ❌ BROKER 1 CRASHES NOW ❌                              │
└─────────────────────────────────────────────────────────────────┘
```

---

## Understanding Key Offsets

Before we dive into failures, let's be crystal clear on these terms:

### LEO (Log End Offset)

```
Definition: The offset of the NEXT message to be written
           (or the offset after the last message)

Leader LEO: 1005
  → Last message is at offset 1004
  → Next message will be at offset 1005

Follower LEO: 1000
  → Last message is at offset 999
  → Next message will be at offset 1000
  → This follower is 5 messages behind leader
```

### HWM (High Water Mark)

```
Definition: The smallest LEO among all ISR members
           (The last offset that ALL ISR replicas have)

Calculation:
  Leader LEO: 1005
  Follower 1 LEO: 1000
  Follower 2 LEO: 1000

  HWM = min(1005, 1000, 1000) = 1000

Importance:
  - Only messages BELOW HWM are visible to consumers
  - Guarantees consumers only see replicated data
  - Ensures consistency even if leader fails

Visual:
  Offset:   0 ───────────────────────── 1000 ─── 1005
            │                             │      │
            │◄── Consumers can read ─────▶│      │
            │                             │      │
            LSO                          HWM    LEO
         (start)                     (replicated)(end)
```

### Why This Matters

```
Messages 0-999:
  ✓ Below HWM
  ✓ Replicated to all ISR members
  ✓ Visible to consumers
  ✓ Safe from data loss (exist on multiple brokers)

Messages 1000-1004:
  ✗ Above HWM
  ✗ Only on leader
  ✗ NOT visible to consumers (yet)
  ⚠️  At risk if leader fails NOW
```

---

## Leader Failure Timeline

### Step-by-Step: What Actually Happens

```
TIME T=0: BROKER 1 CRASHES
┌─────────────────────────────────────────────────────────────────┐
│  Broker 1 (Leader): ✗✗✗ CRASH ✗✗✗                              │
│                                                                 │
│  - Hardware failure / OOM / Network partition / Process kill    │
│  - All messages 1000-1004 exist ONLY on this dead broker        │
│  - Producer trying to send gets NetworkException                │
│  - Consumers trying to fetch get NetworkException               │
└─────────────────────────────────────────────────────────────────┘

TIME T=0 to T=6s: NOBODY KNOWS YET
┌─────────────────────────────────────────────────────────────────┐
│  Controller (Broker 4, let's say):                              │
│    - Waiting for heartbeat from Broker 1                        │
│    - Heartbeat expected every 3 seconds                         │
│    - Timeout configured: 6 seconds (default)                    │
│    - Still within timeout window...                             │
│                                                                 │
│  Broker 2 (Follower):                                           │
│    - Tries to fetch from Broker 1                               │
│    - Gets connection error                                      │
│    - Retries...                                                 │
│                                                                 │
│  Broker 3 (Follower):                                           │
│    - Tries to fetch from Broker 1                               │
│    - Gets connection error                                      │
│    - Retries...                                                 │
│                                                                 │
│  Producers:                                                     │
│    - Trying to send to Broker 1                                 │
│    - Getting connection errors                                  │
│    - Retrying (if configured)                                   │
│    - Requests queuing up in buffer                              │
│                                                                 │
│  Consumers:                                                     │
│    - Trying to fetch from Broker 1                              │
│    - Getting connection errors                                  │
│    - Retrying...                                                │
│                                                                 │
│  STATUS: PARTITION UNAVAILABLE (nobody can read or write)       │
└─────────────────────────────────────────────────────────────────┘

TIME T=6s: CONTROLLER DETECTS FAILURE
┌─────────────────────────────────────────────────────────────────┐
│  Controller:                                                    │
│    1. Broker 1 heartbeat timeout exceeded (6 seconds)           │
│    2. Mark Broker 1 as DOWN                                     │
│    3. Identify affected partitions where Broker 1 is leader:    │
│       - orders-0                                                │
│       - payments-1                                              │
│       - customers-2                                             │
│       - (etc... could be hundreds of partitions)                │
│    4. Start leader election process for each partition          │
│                                                                 │
│  ⚙️  LEADER ELECTION BEGINS ⚙️                                  │
└─────────────────────────────────────────────────────────────────┘

TIME T=6s to T=6.1s: LEADER ELECTION FOR orders-0
┌─────────────────────────────────────────────────────────────────┐
│  Controller's election process:                                 │
│                                                                 │
│  Step 1: Get current state                                      │
│    Current ISR: [1, 2, 3]                                       │
│    Current Leader: 1 (DEAD)                                     │
│                                                                 │
│  Step 2: Remove dead broker from ISR                            │
│    New ISR: [2, 3]                                              │
│                                                                 │
│  Step 3: Select new leader                                      │
│    Algorithm: First replica in ISR that is alive                │
│    Preference order: [2, 3]                                     │
│    New Leader: 2 (Broker 2) ✓                                  │
│                                                                 │
│  Step 4: Increment leader epoch                                 │
│    Old epoch: 5                                                 │
│    New epoch: 6                                                 │
│    (Used to reject stale requests)                              │
│                                                                 │
│  Step 5: Update metadata                                        │
│    Leader: 2                                                    │
│    ISR: [2, 3]                                                  │
│    Epoch: 6                                                     │
│    Partition state: ONLINE                                      │
│                                                                 │
│  Step 6: Send LeaderAndIsrRequest                               │
│    To Broker 2: "You are now leader for orders-0"              │
│    To Broker 3: "New leader for orders-0 is Broker 2"          │
│                                                                 │
│  Step 7: Send UpdateMetadataRequest                             │
│    To ALL brokers: "orders-0 leader is now Broker 2"           │
│    (So producers/consumers can redirect)                        │
└─────────────────────────────────────────────────────────────────┘

TIME T=6.1s: NEW LEADER TAKES OVER
┌─────────────────────────────────────────────────────────────────┐
│                    PARTITION: orders-0                          │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              BROKER 2 (NEW LEADER) ★                     │  │
│  │                                                          │  │
│  │  Offsets: 0 ────────────────────────────────────▶ 1000  │  │
│  │           [============ MESSAGES =============]          │  │
│  │                                                          │  │
│  │  LEO: 1000                                               │  │
│  │  HWM: 1000 (same as LEO now, since fully caught up)     │  │
│  │                                                          │  │
│  │  Actions taken:                                          │  │
│  │  1. Accept produce requests                              │  │
│  │  2. Accept fetch requests                                │  │
│  │  3. Start tracking follower progress                     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              BROKER 3 (FOLLOWER)                         │  │
│  │                                                          │  │
│  │  Offsets: 0 ────────────────────────────────────▶ 1000  │  │
│  │           [============ MESSAGES =============]          │  │
│  │                                                          │  │
│  │  LEO: 1000                                               │  │
│  │  Now fetching from Broker 2 (new leader)                │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ISR: [2, 3]                                                    │
│  Leader: Broker 2                                               │
│  Leader Epoch: 6                                                │
│                                                                 │
│  ⚠️  CRITICAL OBSERVATION:                                      │
│  Messages 1000-1004 from old leader are LOST                    │
│  They never made it to any follower                             │
│  They were not visible to consumers (above HWM)                 │
└─────────────────────────────────────────────────────────────────┘

TIME T=6.1s+: CLIENTS RECOVER
┌─────────────────────────────────────────────────────────────────┐
│  Producers:                                                     │
│    1. Get metadata refresh showing new leader                   │
│    2. Reconnect to Broker 2                                     │
│    3. Retry buffered messages (if retries configured)           │
│    4. Resume sending to Broker 2                                │
│                                                                 │
│  Consumers:                                                     │
│    1. Get metadata refresh showing new leader                   │
│    2. Reconnect to Broker 2                                     │
│    3. Continue fetching from last committed offset              │
│    4. No messages lost from consumer perspective                │
│       (They never saw 1000-1004 anyway)                         │
│                                                                 │
│  TOTAL DOWNTIME: ~100ms (from T=6s to T=6.1s)                   │
│    - Time for election and metadata propagation                 │
│    - Actual detection took 6 seconds (heartbeat timeout)        │
└─────────────────────────────────────────────────────────────────┘
```

### Visual Timeline

```
T=0s      Leader crashes, messages 1000-1004 lost
          ❌ Broker 1 dies
          ▼
T=0-6s    Detection period (heartbeat timeout)
          ⏳ Waiting...
          ⏳ Partition UNAVAILABLE
          ⏳ Producers/consumers retrying
          ▼
T=6s      Controller detects failure
          🔍 Heartbeat timeout
          ▼
T=6-6.1s  Leader election
          ⚙️  Select new leader
          ⚙️  Update metadata
          ⚙️  Notify brokers
          ▼
T=6.1s+   Partition back online
          ✓ Broker 2 is new leader
          ✓ Producers/consumers reconnect
          ✓ Normal operation resumed

EFFECTIVE DOWNTIME: ~100ms (election)
TOTAL UNAVAILABLE: ~6.1 seconds (including detection)
```

---

## Impact of Different Configurations

### Configuration 1: acks=0 (Fire and Forget)

```
Producer Config:
  acks = 0

Behavior:
  Producer ──── Message ────▶ Leader
  Producer doesn't wait for any acknowledgment
  Returns success immediately

SCENARIO: Leader fails right after receiving message

Timeline:
  T=0:   Producer sends message
  T=1ms: Message received by leader's network buffer
  T=1ms: Producer considers it "sent" (returns success)
  T=2ms: ❌ Leader crashes before writing to log

Result:
  ✗ Message LOST (never made it to log)
  ✓ Producer thinks it was successful
  ✗ No retry (producer already moved on)

Risk Level: 🔴 HIGHEST
Data Loss: Messages in network buffers or not yet written to disk

When to use: Metrics, logs, non-critical data where loss is acceptable
```

### Configuration 2: acks=1 (Leader Acknowledgment)

```
Producer Config:
  acks = 1
  retries = 3

Behavior:
  Producer ──── Message ────▶ Leader
                              Leader writes to log
  Producer ◄─── Ack ─────────┤
  (Leader acknowledges BEFORE replication)

SCENARIO: Leader acknowledges, then crashes before replication

Timeline:
  T=0:    Producer sends message (offset 1000)
  T=10ms: Leader writes to local log
  T=11ms: Leader sends acknowledgment to producer
  T=12ms: Producer receives ack (considers successful)
  T=15ms: ❌ Leader crashes before followers fetch
  T=6s:   New leader elected (Broker 2)
  T=6s:   Broker 2 has messages 0-999 only

Result:
  ✗ Message 1000 LOST (was only on leader)
  ✓ Producer thinks it was successful
  ✗ No retry (already acked)
  ✓ Consumer never saw it (was above HWM)

Visual:
  Before crash:
    Leader:    [0...999][1000] ← Message 1000 acknowledged
    Follower:  [0...999]       ← Hasn't replicated yet

  After election:
    New Leader: [0...999]      ← Message 1000 gone forever

Risk Level: 🟡 MEDIUM
Data Loss: Messages acknowledged but not yet replicated

When to use: Balanced approach, most common in production
Note: Acceptable if you can tolerate losing messages between
      acknowledgment and replication (typically milliseconds)
```

### Configuration 3: acks=all with min.insync.replicas=1 (Weak)

```
Producer Config:
  acks = all

Broker Config:
  min.insync.replicas = 1  ← Only leader needs to ack
  replication.factor = 3

Behavior:
  Same as acks=1 when all replicas are in ISR

Problem:
  With min.insync.replicas=1, "all" means "just the leader"
  Doesn't provide any additional safety over acks=1

Timeline:
  T=0:  Producer sends message
  T=5ms: Leader writes to log (LEO=1001)
  T=6ms: Leader sends ack (min.insync.replicas=1 satisfied)
  T=7ms: ❌ Leader crashes
  T=6s: New leader elected, message lost

Result:
  ✗ Same data loss as acks=1
  ✗ Slower (acks=all has overhead)
  ✗ False sense of security

Risk Level: 🟡 MEDIUM (same as acks=1)
Data Loss: Yes, despite acks=all

When to use: ❌ DON'T USE THIS COMBINATION
Recommendation: If using acks=all, set min.insync.replicas >= 2
```

### Configuration 4: acks=all with min.insync.replicas=2 (Strong)

```
Producer Config:
  acks = all
  retries = 3

Broker Config:
  min.insync.replicas = 2
  replication.factor = 3

Behavior:
  Producer ──── Message ────▶ Leader
                              Leader writes to log
                              Leader replicates to followers
                              Wait for 1 follower ack (2 total including leader)
  Producer ◄─── Ack ─────────┤

SCENARIO: Leader crashes after replication

Timeline:
  T=0:    Producer sends message (offset 1000)
  T=5ms:  Leader writes to local log (LEO=1001)
  T=10ms: Leader sends to followers
  T=15ms: Broker 2 writes to log (LEO=1001)
  T=15ms: Broker 2 sends ack to leader
  T=16ms: Leader sends ack to producer (min.insync.replicas=2 satisfied)
  T=17ms: Producer receives ack
  T=20ms: Broker 3 writes to log (LEO=1001) [slower]
  T=25ms: ❌ Leader crashes
  T=6s:   New leader elected (Broker 2)
  T=6s:   Broker 2 has message 1000 ✓

Result:
  ✓ Message 1000 SAFE (on at least 2 brokers)
  ✓ Producer knows it was successful
  ✓ Consumer can eventually read it
  ✓ No data loss

Visual:
  Before crash:
    Leader:     [0...999][1000] ← Acknowledged
    Follower 1: [0...999][1000] ← Replicated ✓
    Follower 2: [0...999][1000] ← Replicated ✓

  After election:
    New Leader: [0...999][1000] ← Message preserved!

Risk Level: 🟢 LOW
Data Loss: Minimal (only if multiple brokers fail simultaneously)

When to use: ✅ RECOMMENDED for critical data
Trade-off: Slightly higher latency (~10-50ms more)
```

### Configuration 5: acks=all with min.insync.replicas=2 AND leader fails mid-replication

```
SCENARIO: Leader crashes AFTER 1 follower acks but BEFORE producer ack

Timeline:
  T=0:    Producer sends message (offset 1000)
  T=5ms:  Leader writes to log (LEO=1001)
  T=10ms: Leader replicates to followers
  T=15ms: Broker 2 writes and acks ✓
  T=16ms: ❌ Leader crashes (BEFORE sending ack to producer)
  T=17ms: Producer timeout (doesn't receive ack)
  T=6s:   New leader elected (Broker 2)
  T=6s:   Producer retries
  T=6.1s: Message already exists (idempotent producer deduplicates)

Result:
  ✓ Message 1000 exists on Broker 2
  ⚠️  Producer doesn't know if it succeeded
  ✓ Producer retries (idempotent producer prevents duplicates)
  ✓ No data loss
  ✓ No duplicates (with enable.idempotence=true)

Risk Level: 🟢 LOW
Behavior: At-least-once delivery (or exactly-once with idempotence)
```

### Configuration 6: replication.factor=1 (No Replication)

```
Broker Config:
  replication.factor = 1  ← Only one copy exists

State:
  Only Broker 1 has the partition
  No followers
  No redundancy

Timeline:
  T=0:  Broker 1 has messages 0-1004
  T=1:  ❌ Broker 1 crashes
  T=6s: Controller detects failure
  T=6s: No other replicas exist
  T=6s: ⚠️  PARTITION OFFLINE

Result:
  ✗ ALL messages 0-1004 UNAVAILABLE
  ✗ Partition cannot come online
  ✗ Must wait for Broker 1 to recover
  ✗ If disk corrupted: ALL DATA LOST

Risk Level: 🔴 CATASTROPHIC
Data Loss: Everything if broker doesn't recover

When to use: ❌ NEVER in production
            ✓ Only for development/testing
```

### Configuration 7: unclean.leader.election.enable=true

```
Broker Config:
  replication.factor = 3
  min.insync.replicas = 2
  unclean.leader.election.enable = true  ← Allow non-ISR as leader

SCENARIO: ALL ISR members are down, only non-ISR replica available

Setup:
  Broker 1 (Leader):    [0...1000] (DEAD)
  Broker 2 (ISR):       [0...1000] (DEAD)
  Broker 3 (non-ISR):   [0...850]  (ALIVE, lagging)

Without unclean election (default):
  - Partition stays OFFLINE
  - Wait for Broker 1 or 2 to recover
  - No data loss, but unavailable
  - Prioritizes consistency

With unclean election (enabled):
  - Broker 3 elected as leader (even though lagging)
  - Partition comes online
  - Messages 851-1000 LOST FOREVER
  - Prioritizes availability

Timeline:
  T=0:   Brokers 1 and 2 crash
  T=6s:  Controller detects failures
  T=6s:  ISR = [3] (only Broker 3 left)
  T=6s:  Broker 3 promoted to leader (unclean election)
  T=6s:  Partition ONLINE with messages 0-850
  T=7s:  Producer writes offset 851 (reusing lost offsets!)

  ⚠️  Messages 851-1000 from old leader are GONE
  ⚠️  New messages will use same offsets
  ⚠️  Possible data inconsistency if consumers cached old data

Risk Level: 🔴 HIGH
Data Loss: All unreplicated messages

When to use:
  ✓ Systems where availability > consistency
  ✓ Non-critical data (analytics, logs)
  ✗ Financial transactions
  ✗ Critical business data
```

---

## The Critical Question: Unreplicated Messages

### What Happens to Messages on Leader but Not Followers?

**Short Answer:** They are LOST forever.

**Why:**

```
FUNDAMENTAL PRINCIPLE:

Only messages below HWM are "committed"
HWM = lowest LEO among ISR members
If followers don't have it, it's not committed
Uncommitted messages can be lost

This is BY DESIGN for consistency
```

### Detailed Breakdown

```
BEFORE CRASH:
┌────────────────────────────────────────────────────────────────┐
│  Leader:    [0...999][1000,1001,1002,1003,1004]               │
│              ↑                ↑                                │
│              │                │                                │
│              │                └─ LEO = 1005                    │
│              └─ HWM = 1000 (followers have up to 999)          │
│                                                                │
│  Follower 1: [0...999]                                         │
│               LEO = 1000                                       │
│                                                                │
│  Follower 2: [0...999]                                         │
│               LEO = 1000                                       │
└────────────────────────────────────────────────────────────────┘

Status of messages 1000-1004:
  ✗ Not replicated
  ✗ Not visible to consumers (above HWM)
  ⚠️  In limbo - acknowledged (if acks=1) but not committed

AFTER CRASH:
┌────────────────────────────────────────────────────────────────┐
│  Leader: ❌ GONE                                               │
│                                                                │
│  New Leader (former Follower 1): [0...999]                     │
│                                   LEO = 1000                   │
│                                   HWM = 1000                   │
│                                                                │
│  Follower (former Follower 2): [0...999]                       │
│                                 LEO = 1000                     │
└────────────────────────────────────────────────────────────────┘

Messages 1000-1004:
  ✗ Physically gone (only existed on dead broker's disk)
  ✗ No way to recover them
  ✗ New leader starts accepting messages at offset 1000
  ⚠️  Offset 1000 will be REUSED for new message!
```

### Why Can't We Recover Them?

```
Option 1: Wait for old leader to come back?
  ❌ Can't wait indefinitely (availability requirement)
  ❌ Old leader's disk might be corrupted
  ❌ Kafka prioritizes availability over waiting

Option 2: Use those messages from old leader when it returns?
  ❌ No! Old leader realizes it was partitioned
  ❌ Sees new leader epoch (6 vs its old epoch 5)
  ❌ Truncates its log back to HWM
  ❌ Messages 1000-1004 deleted to maintain consistency

Option 3: Keep them as "alternate timeline"?
  ❌ Would break consistency guarantees
  ❌ Consumers would see different data depending on timing
  ❌ Exactly-once semantics would be impossible
```

### What Happens When Old Leader Returns?

```
SCENARIO: Broker 1 comes back online

Current State:
  Broker 2 (Leader): [0...999][1000,1001] ← New messages
  Broker 3 (Follower): [0...999][1000,1001]
  Broker 1 (was leader, just recovered): [0...999][1000,1001,1002,1003,1004]
                                          ↑ Old messages        ↑ Old lost msgs

Recovery Process:

Step 1: Broker 1 rejoins cluster
  - Sends FetchRequest to new leader (Broker 2)
  - Includes its LEO: 1005
  - Includes old leader epoch: 5

Step 2: New leader responds with epoch information
  - Current leader epoch: 6
  - Divergence point: offset 1000
  - "You need to truncate back to 1000"

Step 3: Broker 1 truncates its log
  BEFORE: [0...999][1000,1001,1002,1003,1004]
  AFTER:  [0...999]

  ⚠️  Messages 1000-1004 DELETED by Broker 1 itself!

Step 4: Broker 1 fetches from new leader
  Fetches offsets 1000-1001 (new messages)
  Now: [0...999][1000,1001]

Step 5: Broker 1 joins ISR
  ISR: [2, 3, 1]
  Broker 1 is now a follower

RESULT:
  Old "lost" messages 1000-1004 are permanently deleted
  New messages 1000-1001 are the canonical truth
  All replicas have consistent data
```

### Visual: Message Lifecycle

```
STAGE 1: Message Written to Leader
┌─────────────────────────────────────────┐
│ Producer → Leader → Disk                │
│ Status: UNCOMMITTED                     │
│ Visible: NO                             │
│ Replicated: NO                          │
│ Safe: NO ❌                             │
└─────────────────────────────────────────┘

STAGE 2: Replication In Progress
┌─────────────────────────────────────────┐
│ Leader → Followers (network)            │
│ Status: UNCOMMITTED                     │
│ Visible: NO                             │
│ Replicated: PARTIAL                     │
│ Safe: NO ❌                             │
└─────────────────────────────────────────┘

STAGE 3: All ISR Members Have Message
┌─────────────────────────────────────────┐
│ Leader + Followers (all ISR)            │
│ Status: UNCOMMITTED (still above HWM)   │
│ Visible: NO                             │
│ Replicated: YES                         │
│ Safe: YES ✓                             │
└─────────────────────────────────────────┘

STAGE 4: HWM Advanced
┌─────────────────────────────────────────┐
│ All ISR members acked                   │
│ Status: COMMITTED (below HWM)           │
│ Visible: YES                            │
│ Replicated: YES                         │
│ Safe: YES ✓                             │
└─────────────────────────────────────────┘

CRITICAL POINT:
  Leader can fail in Stages 1, 2, or 3 → Message LOST
  Leader fails in Stage 4 → Message SAFE
```

---

## Producer Behavior During Failure

### Producer Request Lifecycle

```
NORMAL OPERATION:

Producer                        Leader (Broker 1)
   │                                 │
   ├──── ProduceRequest ────────────▶│
   │     (messages, acks=all)        │
   │                                 │
   │                                 ├─ Write to log
   │                                 ├─ Replicate
   │                                 ├─ Wait for ISR acks
   │                                 │
   │◀──── ProduceResponse ───────────┤
   │     (success, offset=1000)      │
   │                                 │
   └─ Continue with next batch       │


DURING LEADER FAILURE:

Producer                        Leader (Broker 1)
   │                                 │
   ├──── ProduceRequest ────────────▶│
   │                                 ✗ CRASH
   │
   │ (waiting for response...)
   │
   │ [request.timeout.ms expires]
   │ (default: 30 seconds)
   │
   │ ❌ TimeoutException
   │
   │─ Check retries remaining
   │  (default: retries = 2147483647)
   │
   │─ Refresh metadata
   │  Who is the leader now?
   │
   │ (Wait for election... ~100ms)
   │
   │─ Metadata refreshed
   │  New leader: Broker 2
   │
   ├──── ProduceRequest ────────────▶ Leader (Broker 2)
   │                                  │
   │◀──── ProduceResponse ────────────┤
   │     (success, offset=1000)       │
   │                                  │
   └─ Continue with next batch        │
```

### Producer Configuration Impact

**Configuration Set 1: No Retries (Dangerous)**

```
Config:
  retries = 0
  acks = 1

Timeline:
  T=0:   Producer sends batch 1
  T=1:   Leader writes to log
  T=2:   Leader sends ack
  T=3:   Producer receives ack
  T=4:   Producer sends batch 2
  T=5:   ❌ Leader crashes
  T=5:   Producer gets NetworkException
  T=5:   retries=0 → Give up immediately

Result:
  ✗ Batch 2 LOST
  ✗ Producer moves on
  ✗ Gap in data stream

When to use: ❌ Almost never
```

**Configuration Set 2: Retries Enabled (Standard)**

```
Config:
  retries = 2147483647 (infinite, default)
  retry.backoff.ms = 100
  request.timeout.ms = 30000
  delivery.timeout.ms = 120000 (2 minutes)
  max.in.flight.requests.per.connection = 5

Timeline:
  T=0:    Producer sends batch 1
  T=10ms: Batch 1 acked ✓
  T=20ms: Producer sends batch 2
  T=30ms: ❌ Leader crashes
  T=30ms: Producer waits for response...
  T=30.1s: Timeout (request.timeout.ms)
  T=30.1s: Retry 1 - refresh metadata
  T=30.2s: Send to new leader (Broker 2)
  T=30.3s: Batch 2 acked ✓

Result:
  ✓ Batch 2 succeeds (after retry)
  ✓ No data loss
  ⚠️  30 second delay for that batch

When to use: ✓ Standard production config
```

**Configuration Set 3: Idempotent Producer (Best)**

```
Config:
  enable.idempotence = true
  (Automatically sets: acks=all, retries=MAX, max.in.flight=5)

Behavior:
  Producer gets unique Producer ID (PID)
  Each message gets sequence number
  Broker tracks: PID + Sequence → Offset mapping

Timeline:
  T=0:    Producer (PID=123) sends message (seq=0)
  T=10ms: Leader writes (offset=1000)
  T=11ms: ❌ Leader crashes before ack
  T=30s:  Request timeout
  T=30s:  Producer retries same message (PID=123, seq=0)
  T=30.1s: New leader checks: "I already have PID=123, seq=0"
  T=30.1s: New leader returns success with offset=1000 (idempotent)

Result:
  ✓ No duplicates
  ✓ Exactly-once to partition
  ✓ Producer doesn't know if first send succeeded, doesn't matter

When to use: ✓ Always enable (default in Kafka 3.0+)
```

### Producer Buffering

```
Producer Internal Buffer:

┌────────────────────────────────────────────────────────────────┐
│                    RECORD ACCUMULATOR                          │
│  (buffer.memory = 32 MB default)                               │
│                                                                │
│  Partition 0: [Batch1: 10 msgs][Batch2: 15 msgs][Batch3: ...]│
│  Partition 1: [Batch1: 8 msgs][Batch2: 12 msgs]              │
│  Partition 2: [Batch1: 20 msgs]                               │
└────────────────────────────────────────────────────────────────┘
             │                       ▲
             │                       │
             │                       └─ New sends go here
             │
             └─ Sender thread pulls batches from here

DURING LEADER FAILURE:

1. Sender thread can't send (leader down)
2. Application keeps calling producer.send()
3. Messages accumulate in buffer
4. Buffer fills up (32 MB)
5. Next producer.send() blocks for max.block.ms (default 60s)
6. If buffer still full after 60s → TimeoutException

Timeline:
  T=0:     Leader fails
  T=0-30s: Requests timing out, retrying
  T=0-30s: New sends buffering in memory
  T=30s:   Buffer full (32 MB)
  T=30s:   producer.send() blocks
  T=30.1s: New leader elected
  T=30.2s: Sender drains buffer to new leader
  T=31s:   Buffer has space, producer.send() unblocks

Risk: If election takes > 60s, producer.send() throws exception
```

### Producer Guarantees Summary

| Configuration | Data Loss Risk | Duplicates Risk | Latency | Use Case |
|--------------|----------------|-----------------|---------|----------|
| acks=0 | High | None | Lowest | Metrics, logs |
| acks=1 | Medium | Possible | Low | Balanced |
| acks=all, min.isr=1 | Medium | Possible | Medium | ❌ Don't use |
| acks=all, min.isr=2 | Low | Possible | Higher | Critical data |
| acks=all, min.isr=2, idempotent | Lowest | None | Higher | ✅ Best practice |

---

## Consumer Behavior During Failure

### Consumer Fetch Process

```
NORMAL OPERATION:

Consumer                        Leader (Broker 1)
   │                                 │
   ├──── FetchRequest ──────────────▶│
   │     (offset=1000)               │
   │                                 │
   │                                 ├─ Check offset exists
   │                                 ├─ Offset below HWM? ✓
   │                                 ├─ Read from disk/cache
   │                                 │
   │◀──── FetchResponse ─────────────┤
   │     (messages 1000-1099)        │
   │                                 │
   │─ Process messages               │
   │─ Commit offset 1100             │
   │                                 │


DURING LEADER FAILURE:

Consumer                        Leader (Broker 1)      New Leader (Broker 2)
   │                                 │                      │
   ├──── FetchRequest ──────────────▶│                      │
   │     (offset=1000)               ✗ CRASH               │
   │                                                        │
   │ (waiting for response...)                             │
   │                                                        │
   │ [request.timeout.ms expires]                          │
   │                                                        │
   │ ❌ NetworkException                                   │
   │                                                        │
   │─ Refresh metadata                                     │
   │  (discovers new leader)                               │
   │                                                        │
   ├──── FetchRequest ─────────────────────────────────────▶│
   │     (offset=1000)                                      │
   │                                                        │
   │◀──── FetchResponse ────────────────────────────────────┤
   │     (messages 1000-1099)                               │
   │                                                        │
   │─ Process messages                                      │
   │─ Commit offset 1100                                    │
```

### Key Consumer Observations

**1. Consumers Never See Uncommitted Messages**

```
Scenario:
  Leader has messages 0-1004
  Followers have messages 0-999
  HWM = 1000
  Leader crashes

Consumer behavior:
  ✓ Consumer last fetched up to offset 999
  ✓ Consumer committed offset 1000 (next to read)
  ❌ Leader fails
  ✓ New leader has up to offset 999
  ✓ Consumer resumes from offset 1000...
  ⚠️  But new leader's next message is ALSO offset 1000 (new msg)

Result:
  ✓ Consumer has no gap in offsets
  ✓ Consumer never knew messages 1000-1004 existed on old leader
  ✓ Seamless from consumer perspective
```

**2. Consumer Committed Offsets Are Safe**

```
Consumer offset storage (in __consumer_offsets topic):
  - Also replicated (typically RF=3)
  - Also has min.insync.replicas
  - Survives broker failures

Scenario:
  Consumer commits offset 500
  ❌ Consumer crashes
  ❌ Broker 1 (partition leader) crashes
  ✓ New consumer instance starts
  ✓ Reads committed offset: 500
  ✓ Resumes from offset 500
```

**3. Consumer Rebalancing During Broker Failure**

```
IF consumer was fetching from failed broker:
  ✓ Consumer refreshes metadata
  ✓ Connects to new leader
  ✓ Continues from last committed offset
  ✓ No rebalance needed (partition assignment unchanged)

IF consumer is part of consumer group:
  ✓ Group coordinator tracks consumer health
  ✓ Consumer still sends heartbeats (to coordinator, not data broker)
  ✓ Fetching failures don't trigger rebalance
  ✓ Consumer auto-retries fetching
```

### Consumer Configuration Impact

**Auto-commit Enabled (Default)**

```
Config:
  enable.auto.commit = true
  auto.commit.interval.ms = 5000 (5 seconds)

Behavior:
  T=0s:   Fetch messages 0-99
  T=0.1s: Process messages 0-99
  T=5s:   Auto-commit offset 100
  T=5.1s: Fetch messages 100-199
  T=5.2s: ❌ Consumer crashes

  T=10s:  New consumer starts
  T=10s:  Reads committed offset: 100
  T=10s:  Resumes from offset 100

Result:
  ✓ No message loss
  ✗ Messages 100-199 might be reprocessed (not committed)
  ⚠️  At-least-once delivery

Risk: Duplicate processing if consumer crashes between fetch and commit
```

**Manual Commit (Safer)**

```
Config:
  enable.auto.commit = false

Code:
  while (true) {
    records = consumer.poll(Duration.ofMillis(100));

    for (record : records) {
      process(record);
    }

    consumer.commitSync();  // Commit after processing
  }

Behavior:
  T=0s:   Fetch messages 0-99
  T=0.1s: Process messages 0-99
  T=0.2s: Commit offset 100 ✓
  T=0.3s: Fetch messages 100-199
  T=0.4s: Process messages 100-150
  T=0.5s: ❌ Consumer crashes (before commit)

  T=10s:  New consumer starts
  T=10s:  Reads committed offset: 100
  T=10s:  Resumes from offset 100

Result:
  ✓ No message loss
  ⚠️  Messages 100-150 reprocessed (were processed but not committed)
  ✓ Better control than auto-commit

When to use: ✓ When processing must be complete before commit
```

**Read Committed (For Transactional Producers)**

```
Config:
  isolation.level = read_committed

Behavior:
  Consumer only sees messages from committed transactions
  Skips aborted transaction messages
  Waits at LSO (Last Stable Offset) instead of HWM

Scenario:
  Offset 100: Message A (committed transaction)
  Offset 101: Message B (transaction in progress)
  Offset 102: Message C (transaction in progress)
  Offset 103: Message D (committed transaction)

  Consumer with read_uncommitted:
    Sees: A, B, C, D (all messages)

  Consumer with read_committed:
    Sees: A (stops here until transaction commits)
    Later (after commit): A, B, C, D

When to use: ✓ With exactly-once semantics (EOS)
```

---

## Data Loss Scenarios

### Scenario 1: acks=1, Leader Fails Immediately

```
Config:
  acks = 1
  replication.factor = 3

Timeline:
  0ms:   Producer sends message M1
  10ms:  Leader writes M1 to disk (offset 1000)
  11ms:  Leader sends ack to producer ✓
  12ms:  Producer receives ack (considers successful)
  13ms:  ❌ Leader fails (before followers replicate)
  6s:    New leader elected
  6s:    New leader doesn't have M1

Result:
  ✗ Message M1 LOST
  ✓ Producer thinks it succeeded
  ✗ Cannot retry (already acked)
  ✓ Consumer never saw it (was above HWM)

Probability: Low (milliseconds window)
Data Loss: 1 message
Impact: Silent data loss

Mitigation: Use acks=all
```

### Scenario 2: acks=all, Both Leader and Follower Fail

```
Config:
  acks = all
  min.insync.replicas = 2
  replication.factor = 3

Timeline:
  0ms:   Producer sends message M1
  10ms:  Leader (B1) writes M1
  15ms:  Follower 1 (B2) writes M1
  16ms:  Leader sends ack to producer ✓
  17ms:  Producer receives ack
  20ms:  Follower 2 (B3) is slow, hasn't written yet
  25ms:  ❌ Leader (B1) fails
  26ms:  ❌ Follower 1 (B2) also fails (correlated failure!)
  6s:    Only Follower 2 (B3) available
  6s:    Follower 2 doesn't have M1
  6s:

  Option A (unclean.leader.election.enable = false):
    6s: Partition goes OFFLINE
    6s: Wait for B1 or B2 to recover
    Result: No data loss, but unavailable

  Option B (unclean.leader.election.enable = true):
    6s: B3 elected as leader (unclean election)
    6s: M1 lost
    Result: Available but data loss

Probability: Very low (correlated failures rare)
Data Loss: Messages not replicated to surviving broker
Impact: Depends on unclean election setting

Mitigation:
  - Rack awareness
  - Higher replication factor (5 instead of 3)
  - Better infrastructure (avoid correlated failures)
```

### Scenario 3: Replication Factor = 1

```
Config:
  replication.factor = 1
  (Only one copy exists)

Timeline:
  0ms:  Broker 1 has messages 0-1000
  1ms:  ❌ Broker 1 fails
  6s:   No other replicas
  6s:   Partition OFFLINE

  Option A: Broker comes back, disk OK
    → All data recovered ✓

  Option B: Broker comes back, disk corrupted
    → ALL DATA LOST ✗

  Option C: Broker never comes back
    → ALL DATA LOST ✗

Probability: 100% (if broker doesn't recover)
Data Loss: EVERYTHING
Impact: Catastrophic

Mitigation: ❌ NEVER use RF=1 in production
```

### Scenario 4: Committed Offsets Lost

```
Config (for __consumer_offsets topic):
  replication.factor = 1  ← Misconfigured!

Setup:
  Consumer Group: order-processors
  Committed offsets stored in __consumer_offsets-0 on Broker 4
  Consumer has processed messages 0-500

Timeline:
  0ms:  Consumer commits offset 501
  1ms:  ❌ Broker 4 fails (and doesn't recover)
  6s:   __consumer_offsets-0 LOST
  7s:   New consumer joins group
  7s:   No committed offset found
  7s:   Defaults to auto.offset.reset = latest
  8s:   Consumer starts from end (offset 1000)

Result:
  ✗ Messages 501-999 SKIPPED
  ✗ Data loss from consumer perspective
  ✓ Messages still in Kafka
  ✗ But consumer doesn't know where it was

Mitigation:
  ✓ Ensure offsets.topic.replication.factor = 3 (default)
  ✓ Set auto.offset.reset = earliest (if tolerate reprocessing)
  ✓ External offset store (database) as backup
```

### Scenario 5: Split Brain (Very Rare)

```
Situation:
  Network partition isolates old leader from cluster
  Old leader doesn't know it's been replaced

Timeline:
  0s:    Broker 1 is leader
  0.5s:  Network partition: B1 isolated from B2, B3, Controller
  1s:    B1 still thinks it's leader (hasn't heard otherwise)
  6s:    Controller (can't reach B1) elects B2 as new leader
  7s:    Producer P1 (on B1's network side) writes to B1 (old leader)
  7s:    Producer P2 (on B2's network side) writes to B2 (new leader)
  8s:    Network partition heals
  8s:    B1 discovers new leader epoch
  8s:    B1 truncates divergent messages

Result:
  ✗ Messages written to B1 during partition are LOST
  ✓ Messages written to B2 (real leader) are safe
  ⚠️  "Split brain" data divergence

Mitigation:
  ✓ Leader epoch mechanism (Kafka has this)
  ✓ Fencing: B1 stops accepting writes when sees higher epoch
  ✓ Idempotent producers (detect and reject duplicates)

Kafka's Protection:
  Modern Kafka (0.11+) has strong protections:
  - Leader epochs prevent split brain
  - Old leader rejects requests when it sees higher epoch
  - Very rare in practice
```

---

## Configuration Best Practices

### For Critical Data (Financial, Orders, User Data)

```yaml
# PRODUCER CONFIG
acks: all
retries: 2147483647  # Infinite retries
retry.backoff.ms: 100
request.timeout.ms: 30000
delivery.timeout.ms: 120000  # 2 minutes total
enable.idempotence: true
max.in.flight.requests.per.connection: 5
compression.type: lz4  # Good balance of speed/ratio

# TOPIC CONFIG
replication.factor: 3  # Minimum
min.insync.replicas: 2  # Must have 2 replicas
unclean.leader.election.enable: false  # No data loss

# CONSUMER CONFIG
enable.auto.commit: false  # Manual commit after processing
isolation.level: read_committed  # If using transactions
auto.offset.reset: earliest  # Reprocess rather than skip

# CLUSTER CONFIG
offsets.topic.replication.factor: 3
transaction.state.log.replication.factor: 3
```

**Guarantees:**
- ✓ No data loss (even with single broker failure)
- ✓ Exactly-once semantics (with transactions)
- ✓ Can tolerate 1 broker failure
- ⚠️  Slightly higher latency (~10-50ms)

### For High-Throughput Non-Critical Data (Logs, Metrics)

```yaml
# PRODUCER CONFIG
acks: 1  # Leader only
retries: 5  # Limited retries
request.timeout.ms: 30000
enable.idempotence: false
compression.type: snappy  # Fast compression
batch.size: 32768  # Larger batches
linger.ms: 10  # Small batching delay

# TOPIC CONFIG
replication.factor: 2  # Reduced redundancy
min.insync.replicas: 1  # Leader only
unclean.leader.election.enable: true  # Availability over consistency

# CONSUMER CONFIG
enable.auto.commit: true
auto.commit.interval.ms: 5000
auto.offset.reset: latest  # Skip old data if needed

# CLUSTER CONFIG
offsets.topic.replication.factor: 3  # Keep offset safety
```

**Guarantees:**
- ⚠️  Some data loss possible
- ✓ Higher throughput
- ✓ Lower latency
- ✓ Better availability

### For Balanced Production Workloads

```yaml
# PRODUCER CONFIG
acks: all
retries: 2147483647
request.timeout.ms: 30000
delivery.timeout.ms: 120000
enable.idempotence: true
compression.type: lz4
batch.size: 16384
linger.ms: 0

# TOPIC CONFIG
replication.factor: 3
min.insync.replicas: 2
unclean.leader.election.enable: false

# CONSUMER CONFIG
enable.auto.commit: false
isolation.level: read_committed
auto.offset.reset: earliest

# CLUSTER CONFIG
offsets.topic.replication.factor: 3
transaction.state.log.replication.factor: 3
replica.lag.time.max.ms: 10000
```

**Guarantees:**
- ✓ No data loss with single failure
- ✓ Good throughput
- ✓ Reasonable latency
- ✓ Exactly-once capable

---

## Summary: Quick Reference

### What Messages Are Lost When Leader Fails?

| Message State | Lost? | Why? |
|---------------|-------|------|
| Below HWM (committed) | ❌ No | Replicated to all ISR members |
| Above HWM, replicated to some ISR | ❌ No | If those replicas become leader |
| Above HWM, only on leader | ✅ Yes | No replicas have it |
| Acknowledged with acks=1, not replicated | ✅ Yes | Producer thinks it's sent but it's lost |
| Acknowledged with acks=all, ISR have it | ❌ No | Safe on multiple brokers |
| In producer buffer, not sent | ❌ No | Producer retries after metadata refresh |

### Consumer Perspective

```
Question: Will consumer see data loss?

Answer: Consumers NEVER see data loss from leader failure

Why:
  1. Consumers only read below HWM
  2. HWM = what all ISR replicas have
  3. If leader fails, new leader has all HWM data
  4. Consumer committed offsets are also replicated
  5. Consumer resumes exactly where it left off

BUT consumers may see:
  ✓ Temporary unavailability (during election, ~100ms)
  ✓ Latency spike (during metadata refresh)
  ✗ Never missing messages
  ✗ Never out-of-order messages (within partition)
```

### Downtime During Leader Failure

```
Component          Downtime            Notes
─────────────────────────────────────────────────────────────
Detection          0-6 seconds         Heartbeat timeout
Election           50-200ms            Usually ~100ms
Metadata Propagation 10-100ms          Clients refresh metadata
Producer Recovery  0-30s               Depends on request timeout
Consumer Recovery  0-30s               Depends on request timeout

Total Unavailability: ~6-7 seconds (including detection)
Effective Downtime: ~100-300ms (just the election)

Modern optimizations:
- KRaft: Faster metadata operations
- Incremental rebalancing: Less consumer impact
- Background metadata refresh: Faster client recovery
```

### Configuration Cheat Sheet

**Maximum Durability (No Data Loss):**
```
acks=all + min.insync.replicas=2 + RF=3
unclean.leader.election.enable=false
enable.idempotence=true
```

**Maximum Availability (Tolerate Data Loss):**
```
acks=1 + min.insync.replicas=1 + RF=2
unclean.leader.election.enable=true
enable.idempotence=false
```

**Balanced (Recommended):**
```
acks=all + min.insync.replicas=2 + RF=3
unclean.leader.election.enable=false
enable.idempotence=true
```

### Key Takeaways

1. **Messages above HWM are at risk** - If leader fails before replication completes, these messages are lost forever.

2. **HWM protects consumers** - Consumers never see unreplicated data, so they never experience "data loss" from their perspective.

3. **Configuration matters immensely** - acks=all with min.insync.replicas=2 prevents data loss. acks=1 can lose data.

4. **Detection time is the bottleneck** - 6 seconds to detect failure, only 100ms for election.

5. **Old leader truncates on return** - When old leader recovers, it deletes divergent messages to maintain consistency.

6. **Idempotent producers are essential** - Enable exactly-once semantics and prevent duplicates during retries.

7. **Unclean election is dangerous** - Only enable for non-critical data where availability > consistency.

8. **Replication Factor = 1 is production suicide** - Never use RF=1 for anything important.

---

**Remember:** Kafka is designed for high availability with configurable durability. The key is understanding the trade-offs and choosing the right configuration for your use case.
