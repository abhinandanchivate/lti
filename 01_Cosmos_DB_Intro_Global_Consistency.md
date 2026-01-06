# Azure Cosmos DB - Part 1: Introduction, Global Distribution & Consistency

## Table of Contents
1. [Introduction & Architecture](#introduction--architecture)
2. [Global Distribution & Multi-Region Writes](#global-distribution--multi-region-writes)
3. [Consistency Levels](#consistency-levels)

---

## 1. Introduction & Architecture

### What is Azure Cosmos DB?

Azure Cosmos DB is Microsoft's globally distributed, multi-model NoSQL database service designed for:
- **Mission-critical applications** requiring high availability
- **Planet-scale applications** with users worldwide
- **Low-latency** data access (single-digit millisecond latency)
- **Elastic scalability** for throughput and storage

### Why Azure Cosmos DB?

**Key Advantages:**
- **Global Distribution**: Replicate data across any number of Azure regions
- **Guaranteed SLAs**: 99.999% availability, low latency, throughput, consistency
- **Multiple APIs**: SQL, MongoDB, Cassandra, Gremlin, Table API
- **Automatic Indexing**: No schema or index management required
- **Turnkey Global Distribution**: Add/remove regions with a click
- **Multiple Consistency Models**: Choose the right balance between consistency and performance

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Azure Cosmos DB Account                   │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                    Database(s)                         │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │              Container(s)                        │  │  │
│  │  │  ┌───────────────────────────────────────────┐  │  │  │
│  │  │  │         Logical Partitions               │  │  │  │
│  │  │  │  ┌────────────────────────────────────┐  │  │  │  │
│  │  │  │  │      Physical Partitions           │  │  │  │  │
│  │  │  │  │  ┌──────────────────────────────┐  │  │  │  │  │
│  │  │  │  │  │  Replica Sets (4 replicas)   │  │  │  │  │  │
│  │  │  │  │  └──────────────────────────────┘  │  │  │  │  │
│  │  │  │  └────────────────────────────────────┘  │  │  │  │
│  │  │  └───────────────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**Hierarchy:**
1. **Account**: Top-level resource, unique DNS endpoint
2. **Database**: Logical namespace for containers
3. **Container**: Schema-agnostic container for items
4. **Items**: JSON documents stored in containers

### When to Use Cosmos DB?

**Ideal Use Cases:**
- Global web and mobile applications
- IoT and telemetry data
- Gaming leaderboards and session stores
- Real-time analytics
- E-commerce catalogs and inventory
- Personalization and recommendations
- Graph databases (social networks)

**Not Ideal For:**
- Simple CRUD applications with low traffic
- Applications requiring complex joins and transactions across partitions
- Batch processing workloads (consider Azure Synapse)
- When cost is the primary concern over performance

---

## 2. Global Distribution & Multi-Region Writes

### What is Global Distribution?

Global distribution allows you to replicate your Cosmos DB data across multiple Azure regions worldwide, providing:
- **Low latency** access for users globally
- **High availability** with automatic failover
- **Read/write scalability** across regions

### Architecture of Global Distribution

```
┌──────────────────────────────────────────────────────────────┐
│                 Cosmos DB Account (Global)                    │
│                                                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │ Region 1    │  │ Region 2    │  │ Region 3    │          │
│  │ (Primary)   │  │ (Read)      │  │ (Read)      │          │
│  │             │  │             │  │             │          │
│  │ [Replica 1] │  │ [Replica 2] │  │ [Replica 3] │          │
│  │ [Replica 2] │  │ [Replica 3] │  │ [Replica 4] │          │
│  │ [Replica 3] │  │ [Replica 4] │  │ [Replica 1] │          │
│  │ [Replica 4] │  │ [Replica 1] │  │ [Replica 2] │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
│         │                │                │                   │
│         └────────────────┴────────────────┘                   │
│              Automatic Synchronization                         │
└──────────────────────────────────────────────────────────────┘
```

### Why Global Distribution?

**Benefits:**
- **Reduced Latency**: Users access data from the nearest region
- **Business Continuity**: Automatic failover during regional outages
- **Data Sovereignty**: Keep data in specific regions for compliance
- **Read Scalability**: Distribute read load across multiple regions
- **Disaster Recovery**: Built-in geo-redundancy

### Multi-Region Writes

**Single-Region Write (Default):**
- One write region (primary)
- Multiple read regions
- Writes go to primary, replicated to read regions
- Lower cost, simpler conflict resolution

**Multi-Region Write:**
- All regions accept writes
- Writes replicated asynchronously to all other regions
- Higher availability for writes
- Requires conflict resolution strategy

**When to Use Multi-Region Writes:**
- Applications with globally distributed users needing low write latency
- High write availability requirements
- Active-active application architectures
- When 99.999% write availability SLA is needed

**When NOT to Use:**
- Write workload is concentrated in one region
- Cost sensitivity (multi-region writes cost more)
- Complex conflict resolution requirements

### How to Configure Global Distribution

**Using Azure CLI:**
```bash
# Add read region
az cosmosdb update \
  --name mycosmosdb \
  --resource-group myRG \
  --locations regionName=eastus failoverPriority=0 isZoneRedundant=false \
  --locations regionName=westus failoverPriority=1 isZoneRedundant=false

# Enable multi-region writes
az cosmosdb update \
  --name mycosmosdb \
  --resource-group myRG \
  --enable-multiple-write-locations true
```

### Conflict Resolution

With multi-region writes, conflicts can occur. Cosmos DB offers three strategies:

**1. Last Write Wins (LWW) - Default**
- Uses `_ts` timestamp property
- Most recent write wins
- Simple, deterministic

**2. Custom - User-Defined Function**
- Write custom JavaScript stored procedure
- Complex business logic
- Executes in the region receiving the read

**3. Custom - Async**
- Conflicts logged to conflicts feed
- Application resolves conflicts asynchronously
- Maximum flexibility

**Example Custom Resolution:**
```javascript
function resolveConflict(incomingItem, existingItem, isTombstone, conflictingItems) {
    // Custom logic - highest price wins
    if (incomingItem.price > existingItem.price) {
        return incomingItem;
    }
    return existingItem;
}
```

---

## 3. Consistency Levels

### What are Consistency Levels?

Consistency levels define the trade-off between:
- **Consistency**: How up-to-date data reads are
- **Availability**: System uptime and responsiveness
- **Latency**: Speed of read operations
- **Throughput**: Number of operations per second

### The Five Consistency Levels

```
Strong ←─────────────────────────────────────────→ Eventual
  │              │               │          │           │
Strong    Bounded Staleness   Session  Consistent  Eventual
                                         Prefix
```

### 1. Strong Consistency

**What:**
- Reads guaranteed to return the most recent committed write
- Linearizability guarantee
- Reads never see uncommitted or partial writes

**How it Works:**
```
Write to Region A → Replicate to all regions → Acknowledge write → Allow reads
```

**When to Use:**
- Financial applications
- Inventory systems
- Auction systems
- Banking transactions
- When data accuracy is critical

**Tradeoffs:**
- **Pros**: Strongest consistency guarantee
- **Cons**: Higher latency, lower availability, requires quorum
- **Availability**: Can't provide availability during outages
- **Latency**: 2x latency compared to eventual

### 2. Bounded Staleness

**What:**
- Reads lag behind writes by at most K versions or T time
- Consistency within bounds: K updates or T time interval
- Outside single region: strong consistency

**Parameters:**
- **K**: Number of versions (operations) - range: 10 to 2,147,483,647
- **T**: Time interval - range: 5 seconds to 1 day

**When to Use:**
- Global applications needing strong consistency within a region
- Applications tolerating slight delays
- Social media feeds with "eventual strong" consistency
- Gaming leaderboards

**Tradeoffs:**
- **Pros**: Predictable staleness, good balance
- **Cons**: More complex to reason about
- **Latency**: ~1.5x eventual consistency

### 3. Session Consistency (DEFAULT)

**What:**
- Guarantees consistency within a client session
- Reads your own writes
- Monotonic reads and writes
- Read-your-writes guarantee

**When to Use:**
- Most applications (default choice)
- User-centric applications
- Shopping carts
- User profiles
- Social media posts
- Any application where user sees their own updates

**Tradeoffs:**
- **Pros**: Best balance of consistency, latency, and throughput
- **Cons**: Only guarantees within a session
- **Latency**: Low, ~1.2x eventual

### 4. Consistent Prefix

**What:**
- Reads never see out-of-order writes
- See prefixes of the write sequence
- No gaps in the sequence

**How it Works:**
```
Writes: A, B, C, D, E
Reads might see: A, AB, ABC, ABCD, ABCDE
Never see: AC, B, ACD (out of order)
```

**When to Use:**
- Social media updates (Twitter-like feeds)
- News feeds
- Comment threads
- When order matters but some delay is acceptable

### 5. Eventual Consistency

**What:**
- Weakest consistency
- No ordering guarantees
- Replicas eventually converge
- Maximum availability and performance

**When to Use:**
- Read-heavy workloads
- Analytics and aggregations
- Non-critical data
- Review counts, view counts
- When performance is critical over consistency

**Tradeoffs:**
- **Pros**: Lowest latency, highest throughput, highest availability
- **Cons**: No consistency guarantees
- **Latency**: Lowest possible

### Comparison Matrix

| Level | Read Guarantee | Latency | Availability | Use Case |
|-------|---------------|---------|--------------|----------|
| Strong | Latest committed | Highest | Lowest | Financial, Inventory |
| Bounded Staleness | Max K/T lag | High | Medium | Gaming leaderboards |
| Session | Your writes | Low | High | User profiles, carts |
| Consistent Prefix | Ordered writes | Low | High | Social feeds |
| Eventual | None | Lowest | Highest | Analytics, counts |

### How to Choose?

**Decision Tree:**
```
Does data accuracy impact business critically?
│
├─ YES → Strong or Bounded Staleness
│         │
│         └─ Need global strong? → Bounded Staleness
│         └─ Single region? → Strong
│
└─ NO → Session, Consistent Prefix, or Eventual
          │
          ├─ Users need to see their writes? → Session
          │
          ├─ Order matters? → Consistent Prefix
          │
          └─ Maximum performance? → Eventual
```

### Consistency Level Override

You can override account-level consistency for specific operations:

```csharp
// Account level: Session
// Override to Eventual for this read
ItemResponse<Product> response = await container.ReadItemAsync<Product>(
    id: "123",
    partitionKey: new PartitionKey("electronics"),
    requestOptions: new ItemRequestOptions
    {
        ConsistencyLevel = ConsistencyLevel.Eventual
    }
);
```

**Rules:**
- Can only relax consistency (Strong → Eventual)
- Cannot strengthen (Eventual → Strong)
- Applies per request

### Best Practices

- **Default to Session**: Provides the best balance for most applications
- **Use Strong sparingly**: Only when absolutely necessary
- **Test different levels**: Measure impact on your workload
- **Monitor staleness**: Track replication lag with Bounded Staleness
- **Document your choice**: Explain consistency level selection
- **Consider regional distribution**: Strong consistency across regions has significant latency
