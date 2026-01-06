# Azure Cosmos DB - Part 2: Partitions, RU/s & Indexing

## Table of Contents
1. [Partitions & Partition Keys](#partitions--partition-keys)
2. [RU/s Provisioning](#rus-provisioning)
3. [Indexing Policies & TTL](#indexing-policies--ttl)

---

## 1. Partitions & Partition Keys

### What is Partitioning?

Partitioning is the mechanism by which Cosmos DB achieves:
- **Horizontal scaling**: Distribute data across multiple servers
- **High throughput**: Parallelize operations
- **Unlimited storage**: Scale beyond single server limits

### Partition Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Container                                 │
│                                                               │
│  Logical Partitions (based on partition key value)          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ /userId  │  │ /userId  │  │ /userId  │  │ /userId  │   │
│  │  = "A"   │  │  = "B"   │  │  = "C"   │  │  = "D"   │   │
│  │          │  │          │  │          │  │          │   │
│  │ Items... │  │ Items... │  │ Items... │  │ Items... │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
│       │              │              │              │         │
│       └──────────────┴──────────────┴──────────────┘         │
│                           ↓                                   │
│  Physical Partitions (managed by Cosmos DB)                  │
│  ┌────────────────────┐  ┌────────────────────┐            │
│  │  Physical Part 1   │  │  Physical Part 2   │            │
│  │  [Logical: A, B]   │  │  [Logical: C, D]   │            │
│  │  Max: 50GB         │  │  Max: 50GB         │            │
│  │  Max: 10K RU/s     │  │  Max: 10K RU/s     │            │
│  └────────────────────┘  └────────────────────┘            │
└─────────────────────────────────────────────────────────────┘
```

### Why Partition Keys Matter?

**Critical Reasons:**
1. **Performance**: Queries within a partition are faster
2. **Cost**: Cross-partition queries consume more RUs
3. **Transactions**: Only supported within a partition
4. **Scaling**: Poor key choice limits scalability
5. **Hot Partitions**: Unbalanced keys cause throttling

### Choosing a Partition Key

**Golden Rules:**

1. **High Cardinality**: Many distinct values
   - ✅ Good: userId (millions of users)
   - ❌ Bad: userType (only 3-4 values)

2. **Even Distribution**: Balanced storage and throughput
   - ✅ Good: orderId (random distribution)
   - ❌ Bad: country (some countries have way more data)

3. **Query Pattern Alignment**: Minimize cross-partition queries
   - ✅ Good: Partition by what you filter most
   - ❌ Bad: Partition key not in frequent queries

4. **Transactional Requirements**: Group related items
   - ✅ Good: All order items under same orderId
   - ❌ Bad: Splitting transactional data across partitions

### Partition Key Selection Examples

**E-commerce Application:**

```javascript
// ❌ BAD: Low cardinality
{
    "id": "product123",
    "partitionKey": "electronics", // Only a few categories
    "name": "Laptop",
    "price": 1299
}

// ✅ GOOD: High cardinality + query pattern
{
    "id": "order123",
    "partitionKey": "user456", // Millions of users
    "items": [...],
    "total": 2599
}

// ✅ BETTER: Composite for even distribution
{
    "id": "product123",
    "partitionKey": "electronics-2024-01", // Category + time period
    "name": "Laptop"
}
```

### Best Practices

**Do's:**
- ✅ Choose partition keys with 1000+ distinct values
- ✅ Analyze your query patterns before choosing
- ✅ Distribute read and write operations evenly
- ✅ Keep related data in same partition for transactions
- ✅ Use composite keys for better distribution
- ✅ Test with production-like data volume
- ✅ Monitor partition-level metrics

**Don'ts:**
- ❌ Use low-cardinality keys (status, type, category alone)
- ❌ Choose keys that create hot partitions
- ❌ Ignore your query patterns
- ❌ Expect to change partition keys later (requires migration)

---

## 2. RU/s Provisioning

### What are Request Units (RUs)?

**Definition**: Request Units (RUs) are the currency of performance in Cosmos DB. They represent a normalized measure of resource consumption including:
- CPU
- Memory
- IOPS (Input/Output Operations Per Second)

**Base Rate**: 1 RU = Reading a 1KB document by its ID and partition key

### RU Cost Examples

| Operation | Typical RU Cost |
|-----------|-----------------|
| Point Read (1KB, by ID + partition key) | 1 RU |
| Point Write (1KB) | 5-6 RUs |
| Query (scan 1KB) | 2-3 RUs |
| Update (1KB) | 5-10 RUs |
| Delete | 5 RUs |
| Cross-partition query (10 partitions) | 10x single partition |

### Three Provisioning Models

### 1. Manual (Provisioned) Throughput

**What:**
- Fixed RU/s allocated to container or database
- Guaranteed throughput
- Predictable cost
- Billed hourly

**When to Use:**
- **Predictable workloads**: Steady traffic patterns
- **High sustained throughput**: Continuously high usage (>30%)
- **Cost optimization**: When you can accurately predict needs
- **Production workloads**: Requiring guaranteed performance

**Configuration:**
```csharp
// Container-level
await database.CreateContainerAsync(
    new ContainerProperties("products", "/category"),
    throughput: 10000 // 10,000 RU/s
);
```

**Cost Example:**
```
Manual: 10,000 RU/s
Cost: ~$0.58/hour × 730 hours = ~$423/month
```

### 2. Autoscale Throughput

**What:**
- Automatically scales RU/s based on usage
- Scales between 10% and 100% of max RU/s
- Instant scaling (no warmup)
- Pay for actual usage

**How it Works:**
```
Set max: 20,000 RU/s
↓
Scales automatically: 2,000 - 20,000 RU/s
↓
Cost: Based on highest RU/s each hour
```

**When to Use:**
- **Variable workloads**: Traffic fluctuates significantly
- **Unpredictable patterns**: Can't forecast demand
- **Spiky workloads**: Sudden bursts of traffic
- **New applications**: Don't know usage patterns yet

**Configuration:**
```csharp
await database.CreateContainerAsync(
    new ContainerProperties("orders", "/userId"),
    ThroughputProperties.CreateAutoscaleThroughput(20000) // Max 20K RU/s
);
```

**Cost Example:**
```
Autoscale: Max 20,000 RU/s
Average usage: 8,000 RU/s (40%)
Cost: ~$0.87/hour × 730 hours × 40% = ~$254/month
(~40% savings vs manual 20K RU/s)
```

### 3. Serverless

**What:**
- No provisioning required
- Pay only for RUs consumed
- Storage-based billing
- Best for intermittent workloads

**Limits:**
- Max 5,000 RU/s per partition
- Max 50 GB per container
- Single region only (no multi-region write)
- No SLA on throughput (only availability)

**When to Use:**
- **Intermittent traffic**: Long idle periods
- **Low volume**: < 1 million RUs per day
- **Development/testing**: Variable, low usage
- **Proof of concepts**: Quick prototypes

**Cost Example:**
```
Serverless pricing: ~$0.25 per million RUs
Usage: 10 million RUs/month
Cost: 10 × $0.25 = $2.50/month + storage
```

### Comparison Matrix

| Feature | Manual | Autoscale | Serverless |
|---------|--------|-----------|------------|
| **RU/s Provisioning** | Fixed | Auto 10-100% | On-demand |
| **Minimum RU/s** | 400 | 400 (40-4000 autoscale) | None |
| **Billing** | Per hour | Per highest RU/s | Per request |
| **Best For** | Steady traffic | Variable traffic | Intermittent |
| **SLA** | Yes (throughput) | Yes (throughput) | No throughput SLA |
| **Multi-region writes** | Yes | Yes | No |
| **Storage limit** | Unlimited | Unlimited | 50 GB |

### How to Calculate Required RU/s

**Step-by-Step:**

```
1. Estimate operations per second:
   - Reads: 100 reads/sec
   - Writes: 50 writes/sec
   - Queries: 20 queries/sec

2. Calculate RUs per operation:
   - Read (1KB): 1 RU
   - Write (1KB): 6 RUs
   - Query (returns 10 items): 20 RUs

3. Calculate total:
   (100 × 1) + (50 × 6) + (20 × 20) = 800 RU/s

4. Add 30% buffer:
   800 × 1.3 = 1,040 RU/s
   Round up to: 1,100 RU/s
```

---

## 3. Indexing Policies & TTL

### What is Indexing in Cosmos DB?

Cosmos DB automatically indexes all properties in all documents by default without requiring schema or index definitions.

### Default Indexing Policy

```json
{
    "indexingMode": "consistent",
    "automatic": true,
    "includedPaths": [
        {
            "path": "/*"  // Index all properties
        }
    ],
    "excludedPaths": [
        {
            "path": "/\"_etag\"/?"  // Exclude system property
        }
    ]
}
```

### Indexing Modes

**1. Consistent (Default)**
- Index updated synchronously with writes
- Queries see latest data immediately
- Higher write latency

**2. None**
- No indexing
- Only point reads by ID + partition key work
- Lowest write cost

### Included vs Excluded Paths

**Optimize for E-commerce:**
```json
{
    "indexingMode": "consistent",
    "automatic": true,
    "includedPaths": [
        {"path": "/category/?"},
        {"path": "/price/?"},
        {"path": "/inStock/?"},
        {"path": "/rating/?"}
    ],
    "excludedPaths": [
        {"path": "/*"},
        {"path": "/description/?"},  // Large text
        {"path": "/reviews/*"}       // Array of reviews
    ],
    "compositeIndexes": [
        [
            {"path": "/category", "order": "ascending"},
            {"path": "/price", "order": "descending"}
        ]
    ]
}
```

### Index Types

**1. Range Index (Default)**
- For equality and range queries
- Supports: WHERE, ORDER BY

**2. Spatial Index**
- For geospatial queries
```json
{
    "includedPaths": [
        {
            "path": "/location/?",
            "indexes": [
                {
                    "kind": "Spatial",
                    "dataType": "Point"
                }
            ]
        }
    ]
}
```

**3. Composite Index**
- For ORDER BY on multiple properties
```json
{
    "compositeIndexes": [
        [
            {"path": "/category", "order": "ascending"},
            {"path": "/price", "order": "descending"}
        ]
    ]
}
```

### Time to Live (TTL)

**What is TTL?**
Time to Live automatically deletes documents after a specified period.

**Container-level TTL:**
```csharp
var containerProperties = new ContainerProperties("sessions", "/userId")
{
    DefaultTimeToLive = 3600  // 1 hour in seconds
};
```

**Document-level TTL:**
```javascript
{
    "id": "session123",
    "userId": "user456",
    "data": "...",
    "ttl": 1800  // 30 minutes (overrides container default)
}
```

**No TTL:**
```javascript
{
    "id": "session123",
    "ttl": -1  // Never expires
}
```

### TTL Use Cases

**1. Session Management**
```javascript
{
    "id": "session_abc",
    "userId": "user123",
    "loginTime": "2024-01-04T10:00:00Z",
    "ttl": 3600  // Expire after 1 hour
}
```

**2. Temporary Cache**
```javascript
{
    "id": "cache_key",
    "data": { /* cached response */ },
    "ttl": 300  // 5 minutes
}
```

**3. Event Logs**
```javascript
{
    "id": "log_xyz",
    "timestamp": "2024-01-04T10:00:00Z",
    "message": "Error occurred",
    "ttl": 2592000  // 30 days
}
```

### Best Practices - Indexing

**General Guidelines:**

1. **Start Conservative**
   - Begin with default indexing
   - Monitor query patterns
   - Optimize based on actual usage

2. **Exclude Large Properties**
   ```json
   {
       "excludedPaths": [
           {"path": "/largeTextField/?"},
           {"path": "/binaryData/?"}
       ]
   }
   ```

3. **Index Only What You Query**
   ```json
   {
       "includedPaths": [
           {"path": "/status/?"},
           {"path": "/createdDate/?"}
       ],
       "excludedPaths": [
           {"path": "/*"}
       ]
   }
   ```

4. **Use Composite Indexes for Common Queries**
   ```sql
   -- If this query is common:
   SELECT * FROM c 
   WHERE c.category = "X" 
   ORDER BY c.price DESC
   
   -- Add composite index
   ```

5. **Monitor Index Metrics**
   - Index size in Azure Portal
   - Write RU consumption
   - Query performance

### Best Practices - TTL

**Do's:**
- ✅ Use for temporary data (sessions, caches, logs)
- ✅ Set container-level default for consistency
- ✅ Override per document when needed
- ✅ Use TTL instead of manual cleanup jobs
- ✅ Monitor deleted document metrics

**Don'ts:**
- ❌ Use very short TTL values (<60 seconds) - creates overhead
- ❌ Rely on exact deletion timing (background process)
- ❌ Use TTL for critical business logic timing
- ❌ Forget TTL consumes RUs during deletion
