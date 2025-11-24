# Kafka Compaction & Retention: Complete Reference Guide

## Table of Contents
1. [Log Compaction Basics](#log-compaction-basics)
2. [How Compaction Works](#how-compaction-works)
3. [Tombstone Messages](#tombstone-messages)
4. [Compaction Timing](#compaction-timing)
5. [Configuration Settings](#configuration-settings)
6. [Common Scenarios & Mistakes](#common-scenarios--mistakes)
7. [Recommendations](#recommendations)
8. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)
9. [Troubleshooting](#troubleshooting)

---

## Log Compaction Basics

### Retention Policies

**Time-Based (delete):**
```
cleanup.policy = "delete"
retention.ms = 604800000  (7 days)

Result: Delete messages older than 7 days
Use case: Time-series data, logs
```

**Key-Based (compact):**
```
cleanup.policy = "compact"

Result: Keep only latest value per key
Use case: Database changelogs, current state
```

**Combined:**
```
cleanup.policy = "compact,delete"
retention.ms = 2592000000  (30 days)

Result: Keep latest per key, but delete after 30 days
Use case: GDPR compliance, state with expiration
```

### When to Use Compaction

**Use compaction when:**
- You need the latest state per entity (user profiles, product catalog)
- Keys represent unique identifiers (user_id, sku, order_id)
- You care about current state, not full history
- Consumers may join late and need to rebuild state

**Example use cases:**
```
Customer addresses: Keep current address, not history
Product catalog: Keep current price/details
Application state: Keep latest configuration
Database CDC: Keep latest row state
```

---

## How Compaction Works

### Partition Structure

```
[======= CLEAN =======][======= DIRTY =======]
   Already compacted      Not yet compacted
   Only latest per key    May have duplicates
```

**Clean section:** Previously compacted, one value per key
**Dirty section:** Written since last compaction, has duplicates

### The Compaction Process

**Step 1: Build In-Memory Map**

Cleaner thread reads dirty section:

```
Dirty section messages:
Offset 100: {key: "user:123", value: "Alice"}
Offset 150: {key: "user:456", value: "Bob"}
Offset 200: {key: "user:123", value: "Alice Updated"}

In-Memory Map (24 bytes per entry):
hash("user:123") → Offset 200  (16-byte hash + 8-byte offset)
hash("user:456") → Offset 150
```

**Memory efficiency:**
- 1 GB segment with 1KB messages = 1 million messages
- Map size = 1 million × 24 bytes = 24 MB
- Very efficient!

**Step 2: Scan Clean Section**

```
Read from clean section:
Offset 50: {key: "user:123", value: "Old Value"}

Check map: hash("user:123") exists → Offset 200 is newer
Action: SKIP this message (don't copy)

Offset 75: {key: "user:789", value: "Charlie"}

Check map: hash("user:789") NOT in map
Action: COPY to replacement segment (still latest value)
```

**Step 3: Replace Segment**

After processing:
- Replacement segment has only latest values per key
- Old segment is swapped and deleted
- One message per key remains

### Visual Example

**Before compaction:**
```
Offset 10: {key: A, value: "v1"}
Offset 20: {key: B, value: "v1"}
Offset 30: {key: A, value: "v2"}
Offset 40: {key: C, value: "v1"}
Offset 50: {key: B, value: "v2"}
Offset 60: {key: A, value: "v3"}  ← Latest for A
```

**After compaction:**
```
Offset 40: {key: C, value: "v1"}
Offset 50: {key: B, value: "v2"}  ← Latest for B
Offset 60: {key: A, value: "v3"}  ← Latest for A
```

Offsets 10, 20, 30 removed (older versions of A and B).

---

## Tombstone Messages

### What Is a Tombstone?

A message with a **key and null value** used to signal deletion:

```java
// Create tombstone
producer.send(new ProducerRecord<>("products", "SKU-123", null));
                                                           ↑ null = tombstone
```

### Important: Tombstones Are Created by YOUR Code

**Key point:** Kafka does NOT automatically create tombstones. They must be explicitly sent by your producer.

```
✗ WRONG: "Kafka will create tombstones based on config"
✓ RIGHT: "Only my producer code creates tombstones by sending null values"
```

### Tombstone Lifecycle

```
Day 1: {key: "SKU-123", value: {price: 100}}
Day 2: {key: "SKU-123", value: null}  ← Tombstone sent

Phase 1 (0-24 hours after tombstone):
  - Tombstone visible to consumers
  - Consumers see it and delete from downstream systems

Phase 2 (Compaction runs after 24h):
  - Older message (Day 1) removed
  - Tombstone also removed (delete.retention.ms expired)
  - Key completely disappears from Kafka

Phase 3 (After tombstone removed):
  - New consumers never see the key
  - No record of deletion exists
```

**Timeline:**
```
10:00 AM: Send tombstone
10:00 AM - Next day 10:00 AM: Tombstone visible (24h window)
Next day 10:00 AM: Compaction runs, tombstone removed
After: Key completely gone
```

### The Critical Window

```
delete.retention.ms = 86400000  (24 hours)

Your consumers have 24 hours to:
1. Read the tombstone
2. Process it
3. Delete from downstream systems

If a consumer is offline for > 24 hours:
  → Misses the tombstone
  → Never knows the key was deleted
  → Data inconsistency!
```

### Tombstones with Different Policies

**cleanup.policy = compact:**
```
Regular messages: Kept forever (latest per key)
Tombstones: Deleted after delete.retention.ms (24h default)
Result: Key disappears after 24h
```

**cleanup.policy = compact,delete:**
```
Regular messages: Deleted after retention.ms (e.g., 7 days)
Tombstones: Deleted after delete.retention.ms (24h - EARLIER!)
Result: Tombstones removed before regular messages
```

**cleanup.policy = delete:**
```
Regular messages: Deleted after retention.ms
Tombstones: Also deleted after retention.ms (treated same as regular)
delete.retention.ms: IGNORED (only applies to compaction)
```

### Avoiding Tombstones: Sentinel Values

If you need deletion signals to persist, use a sentinel value instead:

```java
// Instead of null, use deleted flag
public class Product {
    private String sku;
    private Integer price;
    private boolean deleted;  // ← Mark as deleted
}

// Producer
Product deleted = new Product();
deleted.setSku("SKU-123");
deleted.setDeleted(true);
producer.send("products", "SKU-123", deleted);  // NOT null!

// Consumer
if (product.isDeleted()) {
    database.delete(product.getSku());
} else {
    database.upsert(product.getSku(), product);
}
```

**Advantages:**
- ✓ Message stays in Kafka forever
- ✓ Late consumers see deletion marker
- ✓ Auditable (can see when deleted)

**Disadvantages:**
- ✗ Takes up space (not truly deleted)
- ✗ Must handle in consumer logic

---

## Compaction Timing

### When Does Compaction Actually Happen?

Compaction runs when **ALL** these conditions are met:

**Rule 1: Segment Must Be Closed**
```
Active segment: NEVER compacted (currently being written)
Closed segment: Eligible for compaction
```

**Rule 2: Minimum Age (min.compaction.lag.ms)**
```
Youngest message age >= min.compaction.lag.ms
```

**Rule 3: Dirty Ratio OR Maximum Age**
```
EITHER:
  Dirty ratio >= 50% (default)
OR:
  Youngest message age >= max.compaction.lag.ms
```

### The Complete Decision Flow

```
Compaction Manager checks each partition:

1. Any closed segments?
   NO → Skip
   YES → Continue

2. Find oldest closed segment
   Get youngest message timestamp in segment

3. Message age = NOW - youngest_timestamp

4. Age >= min.compaction.lag.ms?
   NO → Skip
   YES → Continue

5. Check: Dirty ratio >= 50% OR age >= max.compaction.lag.ms?
   NO → Skip
   YES → COMPACT!
```

### Understanding Dirty Ratio

```
Dirty Ratio = (Bytes in dirty section) / (Total partition bytes)

Example:
Clean section: 1 GB (previously compacted)
Dirty section: 1 GB (new messages since last compaction)
Total: 2 GB
Dirty ratio: 1 GB / 2 GB = 50%

→ Triggers compaction (reaches threshold)
```

**Default threshold:** 50% (configurable via `min.cleanable.dirty.ratio`)

### Compaction Timing Examples

**Example 1: Only Dirty Ratio Matters (max = ∞)**
```
Config:
  min.compaction.lag.ms = 0
  max.compaction.lag.ms = ∞
  segment.ms = 86400000 (1 day)

Day 1: Messages written to Segment A
Day 2: Segment A closes

Day 2: Compaction check
  ✓ Segment closed
  ✓ Age > 0
  ? Dirty ratio >= 50%?
    YES → COMPACT
    NO → Never compacts (max = ∞)

Problem: Might never compact if dirty ratio stays below 50%
```

**Example 2: Guaranteed Compaction (finite max)**
```
Config:
  min.compaction.lag.ms = 43200000 (12 hours)
  max.compaction.lag.ms = 259200000 (3 days)
  segment.ms = 86400000 (1 day)

Day 1, 10 AM: Messages written
Day 2, 10 AM: Segment closes

Day 2, 10 PM (12h after close):
  ✓ Age >= 12h
  ? Dirty ratio >= 50%?
    YES → COMPACT NOW
    NO → Wait

Day 3, Day 4: Still checking...

Day 5, 10 AM (3 days after first message):
  ✓ Age >= max.compaction.lag.ms
  → FORCE COMPACT (regardless of dirty ratio)

Result: Compaction guaranteed within 3 days
```

**Example 3: High-Frequency Compaction**
```
Config:
  min.compaction.lag.ms = 3600000 (1 hour)
  max.compaction.lag.ms = 86400000 (1 day)
  segment.ms = 3600000 (1 hour)

10:00 AM: Segment A closes
11:00 AM: Eligible (age >= 1h)
          If dirty ratio >= 50% → Compact
Next day 10:00 AM: Force compact (max = 1 day)

Result: Compaction between 1 hour and 1 day
```

### How Settings Interact

```
min.compaction.lag.ms = 12h
max.compaction.lag.ms = 3d
dirty ratio threshold = 50%

Compaction windows:
|---- 12h ----|--------- 60h ---------|---- forever ---|
   NOT          MAYBE                    FORCED
 ELIGIBLE     (if 50% dirty)          (max reached)
```

**Special cases:**
```
min = max = 1 day:
  → Compaction happens EXACTLY at 1 day

min = 0, max = ∞:
  → Compaction ONLY when dirty ratio >= 50% (might be never)

min = 1h, max = 1d:
  → Compaction can happen after 1h if dirty, guaranteed by 1d
```

---

## Configuration Settings

### Default Values

```
# Core compaction settings
cleanup.policy = "delete"                       # NOT compact by default!
delete.retention.ms = 86400000                  # 24 hours
min.compaction.lag.ms = 0                       # No minimum wait
max.compaction.lag.ms = 9223372036854775807     # Infinite (Long.MAX_VALUE)
min.cleanable.dirty.ratio = 0.5                 # 50% threshold

# Retention settings
retention.ms = 604800000                        # 7 days
retention.bytes = -1                            # Infinite
segment.ms = 604800000                          # 7 days
segment.bytes = 1073741824                      # 1 GB

# Message settings
max.message.bytes = 1048588                     # ~1 MB
message.timestamp.type = "CreateTime"
message.timestamp.difference.max.ms = 9223372036854775807  # Infinite

# Broker-level (not topic-level)
log.cleaner.enable = true
log.cleaner.threads = 1
log.cleaner.dedupe.buffer.size = 134217728      # 128 MB total
log.cleaner.backoff.ms = 15000                  # 15 seconds
```

### Setting Explanations

**cleanup.policy**
```
"delete": Time-based deletion (messages deleted after retention.ms)
"compact": Key-based retention (keep latest per key)
"compact,delete": Both (compact + delete after retention.ms)

Recommended for state: "compact"
```

**delete.retention.ms**
```
How long tombstones are kept before removal
Only applies to compacted topics
Default: 86400000 (24 hours)

If you never send tombstones: This setting does nothing
If you send tombstones: Consider 604800000 (7 days) for more safety
```

**min.compaction.lag.ms**
```
Minimum age before message can be compacted
Gives consumers time to read messages

0: Immediate compaction (aggressive)
3600000 (1h): Wait 1 hour
86400000 (1d): Wait 1 day

Recommended: 3600000 - 86400000 (1h - 1d)
```

**max.compaction.lag.ms**
```
Maximum time before message MUST be compacted
Guarantees compaction even if dirty ratio never hits 50%

∞: Might never compact (AVOID!)
86400000 (1d): Guaranteed daily
604800000 (7d): Guaranteed weekly

Recommended: 259200000 - 604800000 (3-7 days)
CRITICAL: Never leave at infinite!
```

**retention.ms**
```
With cleanup.policy="compact" ONLY: Ignored
With cleanup.policy="delete": Delete after this time
With cleanup.policy="compact,delete": Delete even latest values

For compact-only topics: Set to -1 (infinite, makes intent clear)
Default: 604800000 (7 days)
```

**segment.ms**
```
How often to close current segment and start new one
Also controlled by segment.bytes (whichever comes first)

Active segment is NEVER compacted
Smaller segments = more compaction opportunities

604800000 (7d): Weekly segments (default)
86400000 (1d): Daily segments (good balance)
3600000 (1h): Hourly segments (aggressive)
```

---

## Common Scenarios & Mistakes

### Scenario 1: "I Sent a Null Value by Accident"

```
Config: cleanup.policy = "compact"

What happens:
1. Tombstone created for that key
2. After 24h (delete.retention.ms), tombstone removed
3. Key completely disappears from Kafka
4. Cannot be recovered without backup

Prevention:
// In producer code
if (value == null) {
    throw new IllegalArgumentException("Cannot send null!");
}
```

### Scenario 2: "My Topic Is Growing But Never Compacting"

```
Config:
  cleanup.policy = "compact"
  max.compaction.lag.ms = ∞  ← Problem!

Reason:
  - Dirty ratio never reaches 50%
  - max is infinite, so no forced compaction
  - Duplicates accumulate forever

Solution:
  Set max.compaction.lag.ms = 604800000 (7 days)
  → Guarantees weekly compaction
```

### Scenario 3: "Consumer Missed Deletion Signal"

```
Day 1: Send tombstone {key: "user:123", value: null}
Day 1-2: Consumer A sees it, deletes from DB ✓
Day 1-2: Consumer B is OFFLINE
Day 2: Tombstone removed (24h expired)
Day 3: Consumer B comes online, misses tombstone ✗

Result: Consumer B has stale data

Solutions:
1. Increase delete.retention.ms to 604800000 (7 days)
2. Use sentinel value instead of null
3. Track deletions in separate topic
```

### Scenario 4: "Wrong Config - Used compact,delete by Accident"

```
Intended: cleanup.policy = "compact"
Actual: cleanup.policy = "compact,delete"
        retention.ms = 604800000 (7 days)

What happens:
- Regular messages: Compacted AND deleted after 7 days
- Even latest values deleted after 7 days!
- You lose data!

Timeline:
Day 1: {key: "SKU-123", value: {price: 100}}
Day 8: Message deleted (7 days old)
Day 9: SKU-123 completely gone (even though it was latest value)

Fix: Set cleanup.policy = "compact" (only)
```

### Scenario 5: "Null Value in Stream = Corrupt Data?"

```
Question: "Will I get null values in my stream?"

Answer: ONLY if you explicitly send them!

Kafka does NOT send nulls due to:
  ✗ Configuration settings
  ✗ Retention policies
  ✗ Compaction

Nulls appear ONLY when:
  ✓ Your producer sends null value
  ✓ Another producer sends null value

Always defend against this in consumer:
if (value == null) {
    // Handle as deletion signal or error
}
```

### Scenario 6: "Why Is retention.ms Set on Compact Topic?"

```
Config:
  cleanup.policy = "compact"
  retention.ms = 86400000  ← Why is this here?

Answer:
  - With compact ONLY: retention.ms is ignored
  - But it's confusing for future maintainers
  - Might suggest messages deleted after 1 day

Best practice: retention.ms = -1 (explicit infinite)
```

### Scenario 7: "Two Week Data Loss Discovery"

```
Wrong config for 2 weeks:
  cleanup.policy = "delete" (or "compact,delete")
  retention.ms = 604800000 (7 days)

What was lost:
  Week 1 (Day 1-7): All messages still available
  Day 8 onwards: Messages older than 7 days deleted
  Week 2 (when discovered): Only last 7 days of data remains

If tombstones were sent:
  - Tombstones also deleted after retention.ms
  - New consumers don't see deletion signals
  - Data inconsistency in downstream systems

Cannot be recovered without backups
```

---

## Recommendations

### For Topics That Never Send Tombstones (Recommended)

```
topic_config = {
  "cleanup.policy"                      = "compact"
  "delete.retention.ms"                 = "86400000"        # Not used, default is fine
  "max.compaction.lag.ms"               = "259200000"       # 3 days ← CRITICAL
  "min.compaction.lag.ms"               = "43200000"        # 12 hours
  "retention.ms"                        = "-1"              # Infinite ← IMPORTANT
  "segment.ms"                          = "86400000"        # 1 day
  "max.message.bytes"                   = "1048588"
  "message.timestamp.type"              = "CreateTime"
  "message.timestamp.difference.max.ms" = "9223372036854775807"
  "retention.bytes"                     = "-1"
}
```

**Rationale:**
- Compaction guaranteed every 3 days (even if dirty ratio low)
- Wait 12 hours before compacting (batch efficiency)
- Daily segments (good balance)
- Infinite retention (no time-based deletion)

### For High-Frequency Updates (Same Keys Updated Often)

```
topic_config = {
  "cleanup.policy"                      = "compact"
  "max.compaction.lag.ms"               = "86400000"        # 1 day
  "min.compaction.lag.ms"               = "21600000"        # 6 hours
  "retention.ms"                        = "-1"
  "segment.ms"                          = "86400000"        # 1 day
}
```

**Use when:**
- Same SKUs updated frequently
- High duplicate ratio
- Want to reclaim disk space quickly

### For Low-Frequency Updates (Mostly New Keys)

```
topic_config = {
  "cleanup.policy"                      = "compact"
  "max.compaction.lag.ms"               = "604800000"       # 7 days
  "min.compaction.lag.ms"               = "86400000"        # 1 day
  "retention.ms"                        = "-1"
  "segment.ms"                          = "604800000"       # 7 days
}
```

**Use when:**
- Mostly new keys, rare updates
- Low duplicate ratio
- Want to minimize compaction overhead

### For Topics With Tombstones

```
topic_config = {
  "cleanup.policy"                      = "compact"
  "delete.retention.ms"                 = "604800000"       # 7 days (longer!)
  "max.compaction.lag.ms"               = "259200000"       # 3 days
  "min.compaction.lag.ms"               = "3600000"         # 1 hour
  "retention.ms"                        = "-1"
  "segment.ms"                          = "86400000"        # 1 day
}
```

**Key difference:**
- Longer delete.retention.ms (7 days vs 24h)
- Gives consumers more time to see tombstones
- Reduces risk of missed deletion signals

### Critical Changes from Common Mistake Configs

**Must change:**
```
retention.ms: 86400000 → -1
  Reason: Clarify infinite retention, avoid confusion with compact-only

max.compaction.lag.ms: ∞ → 259200000 (3 days)
  Reason: Guarantee compaction runs, prevent unbounded growth
  THIS IS THE MOST CRITICAL CHANGE!
```

**Should change:**
```
min.compaction.lag.ms: 0 → 43200000 (12 hours)
  Reason: More efficient batching, less CPU overhead
```

**Can keep:**
```
segment.ms: 86400000 (1 day)
  Reason: Good balance for most use cases

delete.retention.ms: 86400000 (24h)
  Reason: Standard if not sending tombstones
```

### How to Choose Settings

**1. Update frequency:**
```
Hourly updates to same keys:  min=1h,  max=1d,  segment=1h
Daily updates to same keys:   min=12h, max=3d,  segment=1d
Weekly updates:               min=1d,  max=7d,  segment=7d
```

**2. Duplicate ratio:**
```
High duplicates (same keys updated often): Lower max (compact more often)
Low duplicates (mostly new keys):          Higher max (compact less often)
```

**3. Tombstone usage:**
```
Send tombstones:  delete.retention.ms = 604800000 (7d)
Never send:       delete.retention.ms = 86400000 (24h, default)
```

**4. Consumer lag:**
```
Slow consumers:  Higher min.compaction.lag.ms (give them time)
Fast consumers:  Lower min.compaction.lag.ms (can compact sooner)
```

### Producer Best Practices

**Always validate before sending:**
```java
public void sendProduct(String sku, Product product) {
    // Prevent accidental tombstones
    if (product == null) {
        throw new IllegalArgumentException("Cannot send null product");
    }

    // Prevent null keys (required for compaction)
    if (sku == null || sku.isEmpty()) {
        throw new IllegalArgumentException("SKU cannot be null/empty");
    }

    producer.send(new ProducerRecord<>("products", sku, product));
}
```

**Use sentinel values for deletion:**
```java
public class Product {
    private String sku;
    private String name;
    private Integer price;
    private String status;  // "ACTIVE", "DISCONTINUED", "DELETED"
    private Long updatedAt;
}

public void deleteProduct(String sku) {
    Product deleted = new Product();
    deleted.setSku(sku);
    deleted.setStatus("DELETED");
    deleted.setUpdatedAt(System.currentTimeMillis());
    producer.send("products", sku, deleted);  // NOT null!
}
```

### Consumer Best Practices

**Always handle null values:**
```java
while (true) {
    ConsumerRecords<String, Product> records = consumer.poll(Duration.ofMillis(100));

    for (ConsumerRecord<String, Product> record : records) {
        String key = record.key();
        Product value = record.value();

        if (value == null) {
            // Tombstone detected
            logger.warn("Received tombstone for key: {}", key);
            database.delete(key);
        } else if ("DELETED".equals(value.getStatus())) {
            // Sentinel value deletion
            database.delete(key);
        } else {
            // Normal upsert
            database.upsert(key, value);
        }
    }
}
```

**In Kafka Streams:**
```java
KStream<String, Product> products = builder.stream("products");

// Filter out nulls if you don't want them
KStream<String, Product> validProducts = products
    .filter((key, value) -> value != null);

// Or handle explicitly
products.foreach((key, value) -> {
    if (value == null) {
        System.out.println("DELETE: " + key);
    } else if ("DELETED".equals(value.getStatus())) {
        System.out.println("SOFT DELETE: " + key);
    } else {
        System.out.println("UPSERT: " + key);
    }
});
```

---

## Quick Reference Cheat Sheet

### Compaction Happens When:

```
✓ Segment is closed (not active)
AND
✓ Message age >= min.compaction.lag.ms
AND
✓ (Dirty ratio >= 50% OR Message age >= max.compaction.lag.ms)
```

### Key Formulas:

```
Message age = NOW - message_timestamp
Dirty ratio = dirty_bytes / total_partition_bytes
Compaction map memory = 24 bytes per unique key
```

### Memory Usage:

```
Compaction map = 24 bytes per unique key
1 GB segment (~1M messages) = ~24 MB map memory
```

### Common Time Values:

```
1 hour     = 3600000 ms
6 hours    = 21600000 ms
12 hours   = 43200000 ms
1 day      = 86400000 ms
3 days     = 259200000 ms
7 days     = 604800000 ms
30 days    = 2592000000 ms
Infinite   = 9223372036854775807 ms (Long.MAX_VALUE)
```

### Must-Remember Rules:

1. ✓ **Tombstones are created by YOUR code, not Kafka config**
2. ✓ **Active segment is NEVER compacted**
3. ✓ **With compact-only, retention.ms is ignored (but set to -1 for clarity)**
4. ✓ **max.compaction.lag.ms = ∞ means compaction might never happen**
5. ✓ **Tombstones removed after delete.retention.ms (default 24h)**
6. ✓ **cleanup.policy="compact" must be explicitly set (default is "delete")**
7. ✓ **Compaction based on youngest message age in segment**
8. ✓ **cleanup.policy="compact,delete" deletes even latest values after retention.ms**

### Cleanup Policy Comparison:

```
Policy             Regular Messages              Tombstones                Use Case
==================================================================================
"delete"           Deleted after retention.ms    Deleted after retention.ms   Logs, metrics
"compact"          Kept forever (latest/key)     Deleted after delete.retention.ms   State, CDC
"compact,delete"   Deleted after retention.ms    Deleted after delete.retention.ms   State + expiry
```

---

## Troubleshooting

### Problem: Topic keeps growing, never compacts

```
Symptoms:
- Topic size grows continuously
- Disk space filling up
- Many duplicate keys visible

Check:
1. max.compaction.lag.ms = ∞?
2. Dirty ratio < 50% and never reaches threshold?
3. Log cleaner enabled? (log.cleaner.enable=true)

Fix:
Set max.compaction.lag.ms to finite value:
  max.compaction.lag.ms = 259200000  (3 days recommended)

Verify compaction is running:
  kafka-log-dirs.sh --describe --bootstrap-server localhost:9092
  Check partition size over time
```

### Problem: Consumer got null value unexpectedly

```
Symptoms:
- NullPointerException in consumer
- Unexpected null values in stream
- Data appears to be "missing"

Check:
1. Did your producer send null?
2. Did another producer send null?
3. Check producer logs for errors

Fix:
// Add validation in producer
if (value == null) {
    throw new IllegalArgumentException("Cannot send null!");
}

// Add null handling in consumer
if (value == null) {
    logger.warn("Tombstone for key: {}", key);
    // Handle appropriately
}
```

### Problem: Consumer missed deletion signal

```
Symptoms:
- Downstream system has deleted data
- Kafka topic doesn't have the key
- No deletion event visible

Check:
1. Was consumer offline > delete.retention.ms?
2. Check consumer lag metrics
3. When was tombstone sent vs when consumer restarted?

Fix:
Option 1: Increase delete.retention.ms
  delete.retention.ms = 604800000  (7 days)

Option 2: Use sentinel values instead
  Use deleted flag instead of null values

Option 3: Separate deletion topic
  Keep deletion events in non-compacted topic
```

### Problem: Old messages deleted despite compact-only config

```
Symptoms:
- Messages disappearing after certain time
- "Latest" values being deleted
- Losing historical state

Check:
1. cleanup.policy = "compact,delete"?
2. retention.ms set to short value?
3. Check actual topic config:
   kafka-configs.sh --describe --topic your-topic

Fix:
kafka-configs.sh --bootstrap-server localhost:9092 \
  --alter --topic your-topic \
  --add-config cleanup.policy=compact

kafka-configs.sh --bootstrap-server localhost:9092 \
  --alter --topic your-topic \
  --add-config retention.ms=-1
```

### Problem: Compaction runs too frequently (high CPU)

```
Symptoms:
- High CPU usage on broker
- Frequent I/O spikes
- Slow message throughput

Check:
1. min.compaction.lag.ms = 0?
2. Very small segment.ms?
3. Too many cleaner threads?

Fix:
Increase min.compaction.lag.ms:
  min.compaction.lag.ms = 43200000  (12 hours)

Consider larger segments:
  segment.ms = 86400000  (1 day)

Monitor:
  - FetchMessageConversionsPerSec
  - LogCleanerManager metrics
```

### Problem: Don't know if compaction is working

```
How to verify:

1. Check topic config:
   kafka-configs.sh --describe --topic your-topic --all

2. Check log segments:
   kafka-log-dirs.sh --describe --bootstrap-server localhost:9092 \
     --broker-list 0,1,2 --topic-list your-topic

3. Check for duplicate keys:
   kafka-console-consumer.sh --bootstrap-server localhost:9092 \
     --topic your-topic --from-beginning \
     --property print.key=true | sort | uniq -c

4. Monitor metrics:
   - log-cleaner-recopy-percent
   - log-cleaner-frequency
   - max-dirty-percent

5. Check broker logs:
   grep "Log cleaner thread" server.log
```

### Problem: Compaction seems slow or stuck

```
Symptoms:
- Dirty ratio stays high
- Compaction hasn't run in days
- Segment files accumulating

Check:
1. Log cleaner enabled?
   log.cleaner.enable=true

2. Enough cleaner threads?
   log.cleaner.threads (default 1)

3. Enough memory for dedupe buffer?
   log.cleaner.dedupe.buffer.size (default 128MB)

4. Segments too large for buffer?
   If segment > dedupe buffer, can't compact

Fix:
Increase dedupe buffer (broker config):
  log.cleaner.dedupe.buffer.size=268435456  (256MB)

Add more cleaner threads:
  log.cleaner.threads=2

Check broker logs for errors:
  grep "LogCleaner" server.log
```

### Problem: Partition size larger than expected

```
Symptoms:
- Topic using more disk than expected
- Segment files not being deleted
- Old data still present

Check:
1. Compaction lag settings
2. Number of unique keys vs total messages
3. Segment retention

Debug:
# Check actual data
kafka-run-class.sh kafka.tools.DumpLogSegments \
  --files /path/to/segment-file.log \
  --deep-iteration \
  --print-data-log

# Count unique keys
kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic your-topic --from-beginning \
  --property print.key=true | cut -f1 | sort -u | wc -l

Compare: unique keys × avg message size vs actual partition size
Large difference = compaction needed
```

---

## Real-World Example: E-commerce Product Catalog

### Requirements:
- 100,000 products (SKUs)
- Price updates 1000 products/day
- New products: 100/day
- Each message ~1 KB
- Need current state always available
- No need for price history

### Configuration:

```
topic_config = {
  "cleanup.policy"                      = "compact"
  "retention.ms"                        = "-1"              # Keep forever
  "max.compaction.lag.ms"               = "259200000"       # 3 days max
  "min.compaction.lag.ms"               = "43200000"        # 12 hours min
  "segment.ms"                          = "86400000"        # 1 day segments
  "delete.retention.ms"                 = "86400000"        # Not used (no tombstones)
  "min.cleanable.dirty.ratio"           = "0.5"             # 50% default
}
```

### Expected Behavior:

```
Day 1:
  - 100,000 new products = 100 MB
  - Segment closes end of day
  - Clean: 100 MB, Dirty: 0

Day 2:
  - 1000 updates + 100 new = 1.1 MB
  - Clean: 100 MB, Dirty: 1.1 MB
  - Dirty ratio: 1.1/101.1 = 1%
  - No compaction (below 50%, not 3 days old)

Day 3-4:
  - Another 2.2 MB of updates
  - Dirty ratio still < 50%
  - Waiting...

Day 5:
  - Day 2 segment now 3 days old
  - max.compaction.lag.ms reached
  - FORCED COMPACTION runs
  - 1000 duplicate keys removed
  - ~1 MB of duplicates cleaned

Result:
  - Compaction runs every 3 days minimum
  - Disk usage stays near 100 MB (for 100K products)
  - Not 365 MB after a year (without compaction)
```

### With Tombstones (Product Discontinuation):

```java
// When product is discontinued
public void discontinueProduct(String sku) {
    // DON'T send tombstone (product data valuable for analytics)
    // USE sentinel value instead
    Product discontinued = productRepository.findBySku(sku);
    discontinued.setStatus("DISCONTINUED");
    discontinued.setUpdatedAt(System.currentTimeMillis());

    producer.send(new ProducerRecord<>("products", sku, discontinued));

    // Downstream consumers can:
    // - Remove from search index
    // - Keep in database for order history
    // - Flag in analytics
}
```

---

## Key Takeaways

### The Five Most Important Things:

1. **Set max.compaction.lag.ms to finite value**
   ```
   max.compaction.lag.ms = 259200000  (3 days)
   Never leave at infinite!
   ```

2. **Set retention.ms = -1 for compact-only topics**
   ```
   retention.ms = -1  (not 7 days)
   Makes infinite retention explicit
   ```

3. **Tombstones are explicit, not automatic**
   ```
   Only created when you send null values
   Consider sentinel values instead
   ```

4. **Always handle null values in consumers**
   ```java
   if (value == null) { /* handle tombstone */ }
   ```

5. **Understand the difference between policies**
   ```
   "compact"         = Keep latest, forever
   "delete"          = Delete after time
   "compact,delete"  = Keep latest, but delete after time (RISKY!)
   ```

### Common Mistakes to Avoid:

- ✗ Leaving max.compaction.lag.ms = ∞
- ✗ Setting retention.ms on compact-only topics
- ✗ Using cleanup.policy="compact,delete" without understanding implications
- ✗ Not handling null values in consumers
- ✗ Assuming Kafka creates tombstones automatically
- ✗ Not monitoring compaction metrics
- ✗ Setting min.compaction.lag.ms = max.compaction.lag.ms (no window for early compaction)

### Best Practices Summary:

1. ✓ Always set cleanup.policy="compact" explicitly
2. ✓ Set max.compaction.lag.ms to finite value (3-7 days)
3. ✓ Set retention.ms=-1 for clarity
4. ✓ Use sentinel values instead of tombstones when possible
5. ✓ Validate null values in producer and consumer
6. ✓ Monitor dirty ratio and compaction frequency
7. ✓ Size dedupe buffer appropriately (128MB+ for large topics)
8. ✓ Choose segment.ms based on update frequency
9. ✓ Document your configuration choices for future maintainers

---

**Last Updated:** Based on discussion covering Kafka compaction, retention policies, tombstone handling, and configuration tuning.

**Reference:** Kafka: The Definitive Guide (Second Edition) - Pages 254-261 and related topics.
