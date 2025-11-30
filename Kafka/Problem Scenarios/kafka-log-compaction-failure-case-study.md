# Kafka Log Compaction Failure: A Production Case Study
## When 360 Million Records Became 1.5 Billion - A 4-Year Silent Failure

---

## Executive Summary

This document presents a real-world case study of a catastrophic log compaction failure in Apache Kafka that went undetected for 4 years. A topic configured for log compaction accumulated **1.5 billion messages** when it should have maintained only **360 million unique records**, resulting in a 4.2x data amplification and severe operational impacts on new consumers.

### Key Metrics
- **Topic Age**: 4 years
- **Expected Records**: 360 million (45M SKUs × 8 banners)
- **Actual Records**: 1.57 billion
- **Excess Data**: 1.21 billion obsolete messages
- **Dirty Ratio**: 77% (should trigger compaction at 50%)
- **Consumer Startup Lag**: 1.5 billion messages
- **Log Start Offset**: 0 (proving no segments ever deleted)

---

## Table of Contents
1. [The Problem Statement](#the-problem-statement)
2. [Topic Configuration and Context](#topic-configuration-and-context)
3. [The Investigation Process](#the-investigation-process)
4. [Root Cause Analysis](#root-cause-analysis)
5. [Technical Deep Dive](#technical-deep-dive)
6. [The Solution](#the-solution)
7. [Lessons Learned](#lessons-learned)
8. [Monitoring and Prevention](#monitoring-and-prevention)

---

## The Problem Statement

### Initial Symptom
A new Kafka consumer group was created for the topic `product-forecasted-item-selling-view-operational-avro` with `auto.offset.reset=earliest`. Upon startup, the consumer group immediately showed a lag of approximately **1.5 billion messages**.

### The Business Impact
- **Consumer Startup Time**: New consumers took hours to days to catch up
- **Resource Consumption**: Unnecessary network, CPU, and memory usage processing obsolete data
- **Operational Risk**: Risk of consumer timeouts and rebalancing during initial consumption
- **Storage Cost**: 4.2x more storage than necessary on Kafka brokers

### The Contradiction
The topic was configured for log compaction with the following understanding:
- **Maximum Unique Keys**: 45 million SKUs × 8 banner combinations = 360 million
- **Compaction Policy**: Only the latest message per key should be retained
- **Expected Behavior**: Topic size should stabilize around 360 million messages
- **Actual Behavior**: Topic contained 1.57 billion messages after 4 years

---

## Topic Configuration and Context

### Topic Details
```
Name: product-forecasted-item-selling-view-operational-avro
Partitions: 100
Key Format: sku-banner (e.g., "7751959-NORDSTROM_RACK")
Purpose: Forecasted inventory and selling data for products
```

### Relevant Configuration Parameters
```properties
# Compaction Settings
cleanup.policy=compact
min.cleanable.dirty.ratio=0.5
max.compaction.lag.ms=9223372036854775807  # Effectively infinite
min.compaction.lag.ms=0

# Segment Settings
segment.bytes=104857600  # 100 MB
segment.ms=86400000      # 1 day
segment.jitter.ms=0

# Retention Settings
retention.bytes=-1        # Infinite
retention.ms=86400000     # 1 day (for delete tombstones)
delete.retention.ms=86400000

# Other Settings
compression.type=producer
min.insync.replicas=2
max.message.bytes=1048588
```

### Data Patterns
- **Total SKUs in System**: ~45 million
- **Active SKUs (frequently updated)**: ~10 million
- **Banner Variations**: 8 different combinations
- **Update Frequency**: Daily updates for active SKUs
- **Key Space**: `45M × 8 = 360M` maximum unique keys

---

## The Investigation Process

### Step 1: Initial Observation
```bash
# New consumer group created
kafka-consumer-groups.sh --bootstrap-server <broker> \
  --group new-consumer-group \
  --describe

# Output showed:
# LAG: ~1,500,000,000 messages
```

### Step 2: Verify Topic Configuration
```bash
kafka-configs.sh --bootstrap-server <broker> \
  --entity-type topics \
  --entity-name product-forecasted-item-selling-view-operational-avro \
  --describe

# Confirmed: cleanup.policy=compact
```

### Step 3: Calculate Expected vs Actual Size

#### Expected Size Calculation
```
Maximum Unique Keys = 45,000,000 SKUs × 8 Banners = 360,000,000 keys
With log compaction, topic should contain ≤ 360,000,000 messages
```

#### Actual Size Verification
```bash
# Check Log End Offset (LEO) for all partitions
kafka-run-class kafka.tools.GetOffsetShell \
  --broker-list <broker> \
  --topic product-forecasted-item-selling-view-operational-avro \
  --time -1  # Latest offset

# Results:
# Partition 0: 15,700,000
# Partition 1: 15,680,000
# ... (similar for all 100 partitions)
#
# Total: ~15,700,000 × 100 = 1,570,000,000 messages
```

### Step 4: Check Log Start Offset (The Smoking Gun)
```bash
# Check earliest offset available
kafka-consumer-groups.sh --bootstrap-server <broker> \
  --group temp-test-group \
  --topic product-forecasted-item-selling-view-operational-avro \
  --reset-offsets --to-earliest --dry-run

# Results for all partitions:
# Log Start Offset (LSO): 0
```

**Critical Finding**: LSO of 0 means the very first messages written 4 years ago are still on disk. No segments have ever been deleted.

### Step 5: Calculate Dirty Ratio
```
Total Messages: 1,570,000,000
Expected Unique: 360,000,000
Duplicate/Obsolete: 1,210,000,000

Dirty Ratio = 1,210,000,000 / 1,570,000,000 = 77%
Configuration: min.cleanable.dirty.ratio = 0.5 (50%)

Conclusion: Topic is 27% above the compaction threshold!
```

---

## Root Cause Analysis

### Initial Hypothesis (Incorrect)
Initially suspected that `segment.ms=86400000` (1 day) was causing the issue by creating daily segments that individually never reached the 50% dirty ratio threshold.

**Why This Was Wrong**: The dirty ratio is calculated across ALL closed segments combined, not per individual segment.

### Actual Root Cause: Log Cleaner Resource Starvation

The log cleaner was either:
1. **Dead/Crashed**: Cleaner threads died and were never restarted
2. **Severely Under-resourced**: Insufficient CPU, memory, or I/O allocation
3. **Stuck/Blocked**: Possibly on corrupted segments or memory pressure

### Evidence Supporting Resource Starvation

1. **Massive Offset Map Requirements**
   - 360 million unique keys require substantial memory for the offset map
   - Each entry needs ~24 bytes (8-byte offset + 16-byte MD5 hash)
   - Total: ~8.6 GB just for the offset map

2. **High Processing Load**
   - 100 partitions to clean
   - 1.57 billion messages to process
   - Continuous new writes competing for I/O

3. **Configuration Bottlenecks**
   ```properties
   log.cleaner.threads=1          # Default, likely insufficient
   log.cleaner.io.max.bytes.per.second=∞  # May be throttled
   log.cleaner.dedupe.buffer.size=134217728  # 128MB, too small for 360M keys
   ```

---

## Technical Deep Dive

### Understanding Kafka Offsets

#### Log End Offset (LEO)
- **Definition**: The offset that will be assigned to the next message produced
- **In This Case**: 15.7M per partition × 100 partitions = 1.57B total
- **Behavior**: Always increases, never affected by compaction

#### Log Start Offset (LSO)
- **Definition**: The earliest message offset still retained on disk
- **In This Case**: 0 for all partitions
- **Expected Behavior**: Should advance as old segments are cleaned and deleted
- **Actual Behavior**: Never advanced in 4 years

#### Consumer Lag
- **Definition**: Difference between consumer's current offset and LEO
- **In This Case**: 1.5B messages for new consumers starting at earliest

### Log Compaction Mechanics

#### How Compaction Should Work
1. **Segment Closure**: Active segment rolls based on size or time
2. **Dirty Ratio Check**: Cleaner calculates ratio of duplicate bytes
3. **Cleaning Process**:
   ```
   a. Build offset map (key → latest offset)
   b. Read through segments
   c. Keep only latest message per key
   d. Rewrite cleaned segments
   e. Delete old segments
   ```
4. **LSO Advancement**: Start offset moves forward as old segments deleted

#### Why It Failed Here
```
Expected Flow:
Segments → Dirty Ratio Check (77% > 50% ✓) → Compaction → Clean Segments

Actual Flow:
Segments → Dirty Ratio Check (77% > 50% ✓) → ❌ No Resources → No Compaction
                                                 ↑
                                                 │
                                            The Failure Point
```

### The Compound Effect Over Time

#### Year 1
- Active SKUs: 10M
- Daily updates: 10M × 8 = 80M messages/day
- After 365 days: ~29B messages total
- Should compact to: 360M
- Likely state: Some compaction happening but falling behind

#### Year 2-4
- Cleaner increasingly behind
- Backlog growing exponentially
- Eventually cleaner gives up or crashes
- No monitoring alerts fired
- Silent accumulation continues

---

## The Solution

### Immediate Actions

#### 1. Verify Cleaner Status
```bash
# Check if cleaner threads are alive
jstack <kafka-broker-pid> | grep -i "kafka-log-cleaner"

# Check cleaner metrics
kafka-run-class kafka.tools.JmxTool \
  --object-name kafka.log:type=LogCleanerManager,name=time-since-last-run-ms

# Check for uncleanable partitions
grep -i "marked as uncleanable" /var/log/kafka/server.log
```

#### 2. Increase Cleaner Resources
```bash
# Broker-level settings (requires restart)
log.cleaner.threads=8  # Increase from default 1
log.cleaner.dedupe.buffer.size=536870912  # 512MB, up from 128MB
log.cleaner.io.buffer.size=1048576  # 1MB
log.cleaner.io.max.bytes.per.second=∞  # Remove any throttling

# Topic-level temporary fix
kafka-configs.sh --bootstrap-server <broker> \
  --entity-type topics \
  --entity-name product-forecasted-item-selling-view-operational-avro \
  --alter \
  --add-config min.cleanable.dirty.ratio=0.1,max.compaction.lag.ms=3600000
```

#### 3. Force Immediate Compaction
```bash
# Temporarily set very aggressive compaction
kafka-configs.sh --bootstrap-server <broker> \
  --entity-type topics \
  --entity-name product-forecasted-item-selling-view-operational-avro \
  --alter \
  --add-config min.cleanable.dirty.ratio=0.0

# Monitor progress
watch -n 10 'kafka-log-dirs.sh --describe \
  --bootstrap-server <broker> \
  --topic-list product-forecasted-item-selling-view-operational-avro \
  | grep -E "topic|size"'

# Restore normal settings after compaction
kafka-configs.sh --bootstrap-server <broker> \
  --entity-type topics \
  --entity-name product-forecasted-item-selling-view-operational-avro \
  --alter \
  --add-config min.cleanable.dirty.ratio=0.3
```

### Long-term Solutions

#### Option 1: Dedicated Cleaner Resources
```properties
# Separate cleaner thread pool for high-volume topics
log.cleaner.threads=16
log.cleaner.dedupe.buffer.size=1073741824  # 1GB
log.cleaner.io.buffer.load.factor=0.9
```

#### Option 2: Topic Redesign
```properties
# More partitions for better parallelism
num.partitions=200

# Larger segments for efficiency
segment.bytes=1073741824  # 1GB

# Remove time-based rolling
segment.ms=604800000  # 7 days instead of 1
```

#### Option 3: Tiered Storage (Kafka 2.8+)
```properties
# Move old segments to cheaper storage
remote.storage.enable=true
local.retention.bytes=107374182400  # Keep 100GB locally
remote.retention.bytes=-1  # Unlimited remote
```

---

## Lessons Learned

### 1. Silent Failures Are Dangerous
- **Problem**: Log compaction can fail silently without alerting
- **Solution**: Implement comprehensive monitoring (see next section)

### 2. Default Settings Are Insufficient
- **Problem**: Single cleaner thread cannot handle high-volume topics
- **Solution**: Size cleaner resources based on topic volume and key cardinality

### 3. Dirty Ratio Alone Is Not Enough
- **Problem**: High dirty ratio doesn't guarantee compaction if resources are insufficient
- **Solution**: Monitor actual compaction execution, not just eligibility

### 4. LSO Is a Critical Health Metric
- **Problem**: LSO of 0 after years indicates complete failure
- **Solution**: Alert when LSO doesn't advance for extended periods

### 5. Time-Based Segment Rolling Can Mask Issues
- **Problem**: Daily segments spread updates thin, making issues less obvious
- **Solution**: Prefer size-based rolling for compacted topics

---

## Monitoring and Prevention

### Critical Metrics to Monitor

#### 1. Log Start Offset (LSO) Advancement
```python
# Alert if LSO hasn't advanced in 24 hours
if current_lso == lso_24h_ago:
    alert("Log compaction may be failing for topic X")
```

#### 2. Cleaner Lag Metrics
```bash
# JMX Metrics to track
kafka.log:type=LogCleanerManager,name=time-since-last-run-ms
kafka.log:type=LogCleanerManager,name=max-dirty-percent
kafka.log:type=LogCleaner,name=cleaner-recopy-ratio
kafka.log:type=LogCleanerManager,name=uncleanable-partitions-count
```

#### 3. Topic Size vs Expected Size
```python
# Alert if topic size exceeds expected by 2x
expected_size = num_unique_keys * avg_message_size
actual_size = sum(partition_sizes)
if actual_size > expected_size * 2:
    alert("Topic size exceeds expected: possible compaction failure")
```

### Monitoring Dashboard Example
```yaml
Kafka Compaction Health Dashboard:
  - Panel 1: LSO Advancement Rate (should be > 0)
  - Panel 2: Cleaner Thread CPU Usage (should be active)
  - Panel 3: Dirty Ratio by Topic (highlight > threshold)
  - Panel 4: Topic Size vs Expected Size Ratio
  - Panel 5: Cleaner Last Run Time (should be recent)
  - Panel 6: Uncleanable Partitions Count (should be 0)
```

### Automated Health Checks
```bash
#!/bin/bash
# Daily compaction health check script

TOPIC="product-forecasted-item-selling-view-operational-avro"
BROKER="localhost:9092"

# Check LSO advancement
LSO_NOW=$(kafka-run-class kafka.tools.GetOffsetShell \
  --broker-list $BROKER --topic $TOPIC --time -2 \
  | awk '{sum+=$3} END {print sum}')

LSO_YESTERDAY=$(cat /tmp/lso_yesterday.txt)

if [ "$LSO_NOW" -eq "$LSO_YESTERDAY" ]; then
  echo "CRITICAL: LSO has not advanced in 24 hours!"
  # Send alert
fi

echo $LSO_NOW > /tmp/lso_yesterday.txt

# Check cleaner status
CLEANER_LAG=$(kafka-run-class kafka.tools.JmxTool \
  --object-name kafka.log:type=LogCleanerManager,name=time-since-last-run-ms \
  --one-time | grep Value | awk '{print $2}')

if [ $CLEANER_LAG -gt 3600000 ]; then  # 1 hour
  echo "WARNING: Log cleaner hasn't run in over 1 hour!"
  # Send alert
fi
```

### Prevention Checklist

#### Pre-Production
- [ ] Calculate expected unique keys and size
- [ ] Size cleaner resources appropriately
- [ ] Set up monitoring before going live
- [ ] Test compaction under load

#### Production Operations
- [ ] Weekly review of LSO advancement
- [ ] Monthly review of topic size vs expected
- [ ] Quarterly cleaner resource evaluation
- [ ] Annual compaction efficiency audit

#### Incident Response Plan
1. **Detection**: Alert fires for LSO not advancing
2. **Verification**: Check cleaner thread status and metrics
3. **Immediate Action**: Increase cleaner threads if needed
4. **Recovery**: Force aggressive compaction temporarily
5. **Post-Mortem**: Analyze why compaction fell behind

---

## Conclusion

This case study demonstrates how a seemingly well-configured Kafka topic can silently accumulate years of technical debt when log compaction fails due to resource constraints. The key takeaways are:

1. **Trust but Verify**: Configuration alone doesn't guarantee behavior
2. **Monitor Actively**: LSO and cleaner metrics are critical health indicators
3. **Resource Appropriately**: Default settings rarely work for high-volume topics
4. **Act Quickly**: Compaction backlogs compound exponentially

The 1.5 billion message backlog that should have been 360 million represents not just wasted storage, but operational risk and degraded performance that impacts the entire data pipeline. Regular monitoring and proactive resource management would have prevented this 4-year accumulation.

---

## Appendix: Quick Reference Commands

### Diagnostic Commands
```bash
# Check topic configuration
kafka-configs.sh --describe --entity-type topics --entity-name <topic>

# Check Log End Offset (LEO)
kafka-run-class kafka.tools.GetOffsetShell --time -1

# Check Log Start Offset (LSO)
kafka-run-class kafka.tools.GetOffsetShell --time -2

# Check consumer lag
kafka-consumer-groups.sh --describe --group <group>

# Check cleaner status via JMX
kafka-run-class kafka.tools.JmxTool --object-name kafka.log:type=LogCleanerManager,name=*

# Check partition sizes
kafka-log-dirs.sh --describe --topic-list <topic>
```

### Recovery Commands
```bash
# Force aggressive compaction
kafka-configs.sh --alter --add-config min.cleanable.dirty.ratio=0.0

# Increase cleaner threads (requires broker restart)
# In server.properties: log.cleaner.threads=8

# Monitor compaction progress
watch 'kafka-log-dirs.sh --describe | grep -A5 <topic>'
```

---

*Document Version: 1.0*
*Last Updated: November 2024*
*Case Study: Production Kafka Cluster - 4 Year Compaction Failure*