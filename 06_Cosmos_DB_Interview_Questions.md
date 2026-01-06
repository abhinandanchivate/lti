# Azure Cosmos DB - Part 6: Interview Questions & Answers

## Table of Contents
1. [Basic Level Questions](#basic-level)
2. [Intermediate Level Questions](#intermediate-level)
3. [Advanced Level Questions](#advanced-level)

---

## Basic Level

### Q1: What is Azure Cosmos DB and what are its key features?

**Answer:**
Azure Cosmos DB is Microsoft's globally distributed, multi-model NoSQL database service.

**Key Features:**
- **Global Distribution**: Replicate across multiple Azure regions with turnkey configuration
- **Multi-Model Support**: SQL, MongoDB, Cassandra, Gremlin, Table APIs
- **Guaranteed SLAs**: 99.999% availability, <10ms latency (P99), throughput, consistency
- **Elastic Scalability**: Independently scale storage and throughput
- **Multiple Consistency Levels**: Five levels from strong to eventual
- **Automatic Indexing**: Schema-agnostic without index management

**When to use:**
- Global applications needing low latency worldwide
- Mission-critical apps requiring high availability
- IoT and telemetry with high write throughput
- Real-time analytics and personalization

---

### Q2: Explain the five consistency levels in Cosmos DB.

**Answer:**

**1. Strong**
- Linearizability guarantee - reads always return latest committed write
- Highest latency, can't provide availability during outages
- **Use**: Financial systems, inventory management

**2. Bounded Staleness**
- Reads lag by max K versions or T time
- Predictable staleness with configurable bounds
- **Use**: Gaming leaderboards, global apps with eventual strong needs

**3. Session (Default)**
- Monotonic reads/writes, read-your-writes within client session
- Best balance for most applications
- **Use**: User profiles, shopping carts, social media

**4. Consistent Prefix**
- Reads never see out-of-order writes
- See prefixes of write sequence
- **Use**: Social media feeds, news feeds

**5. Eventual**
- Weakest consistency, highest performance
- No ordering guarantees
- **Use**: Analytics, view counts, non-critical data

**Comparison:**
```
Strong > Bounded Staleness > Session > Consistent Prefix > Eventual
(Consistency ↑, Performance ↓)
```

---

### Q3: What is a partition key and why is it important?

**Answer:**

**Definition:**
A partition key is a property path that determines data distribution across physical partitions.

**Importance:**
1. **Scalability**: Enables horizontal partitioning and unlimited scale
2. **Performance**: Queries within partition are faster (1 RU vs 10+ RUs)
3. **Transactions**: ACID transactions only work within single partition
4. **Cost**: Cross-partition queries consume significantly more RUs
5. **Hot Partitions**: Poor key causes throttling and performance issues

**Best Practices:**
- **High Cardinality**: 1000+ distinct values (userId ✅, userType ❌)
- **Even Distribution**: Balanced storage and traffic (orderId ✅, country ❌)
- **Query Alignment**: Minimize cross-partition queries
- **Transaction Support**: Group related data together

**Example:**
```javascript
// E-commerce order
{
    "id": "order123",
    "partitionKey": "user456",  // userId for user-centric queries
    "items": [...],
    "total": 299.99
}
```

---

### Q4: What are Request Units (RUs)?

**Answer:**

**Definition:**
RUs are normalized measures of resource consumption abstracting CPU, memory, and IOPS.

**Base Unit:** 1 RU = reading 1KB document by ID + partition key

**Typical Costs:**
- Point read (1KB): **1 RU**
- Point write (1KB): **5-6 RUs**
- Query scan (1KB): **2-3 RUs**
- Update (1KB): **5-10 RUs**
- Cross-partition query: **RUs × partitions**

**Factors Affecting Cost:**
- Document size (larger = more RUs)
- Indexing policy (more indexes = higher writes)
- Consistency level (Strong > Eventual)
- Query complexity
- Number of properties returned

**Example Calculation:**
```
100 reads/sec × 1 RU = 100 RU/s
50 writes/sec × 6 RUs = 300 RU/s
20 queries/sec × 20 RUs = 400 RU/s
Total: 800 RU/s + 30% buffer = ~1,100 RU/s needed
```

---

### Q5: What's the difference between manual and autoscale throughput?

**Answer:**

| Feature | Manual | Autoscale |
|---------|--------|-----------|
| **Provisioning** | Fixed RU/s | Max RU/s, scales 10-100% |
| **Cost** | $X/hour fixed | Variable ($1.5X at max, less at low usage) |
| **Scaling** | Manual adjustment | Automatic based on load |
| **Best For** | Steady, predictable traffic | Variable, spiky traffic |
| **Billing** | Hourly rate | Highest RU/s used each hour |

**Decision Tree:**
```
Predictable & steady traffic → Manual
Variable traffic → Autoscale
Very low/intermittent → Serverless
```

**Example:**
```
Manual 10K RU/s: $0.58/hour = $423/month

Autoscale 10K max:
- At 100%: $0.87/hour
- At 40% avg: ~$0.35/hour = $255/month
Savings: $168/month (40%)
```

---

## Intermediate Level

### Q6: How do you design a partition key strategy for an e-commerce application?

**Answer:**

**Analysis Steps:**

**1. Identify Access Patterns:**
- User queries their orders
- Admin views all orders
- Reports by date range
- Order details by orderId

**2. Evaluate Options:**

**Option A: userId**
```javascript
{
    "id": "order123",
    "partitionKey": "user456",
    "items": [...]
}
```
✅ Good: User-centric queries efficient
✅ Good: All user's orders in one partition (transactions)
❌ Risk: High-volume users create hot partitions

**Option B: orderId**
```javascript
{
    "id": "order123",
    "partitionKey": "order123"
}
```
✅ Good: Perfect distribution
❌ Bad: Can't efficiently query by user
❌ Bad: No cross-order transactions

**Option C: Composite (userId + date)**
```javascript
{
    "id": "order123",
    "partitionKey": "user456-2024-01",
    "userId": "user456",
    "orderDate": "2024-01-15"
}
```
✅ Good: Balances distribution
✅ Good: Prevents user hot partitions
✅ Good: Efficient for recent orders
✅ Good: Time-based queries work well

**Recommendation:** Option C for production

**3. Validation:**
- Estimate cardinality: millions of user-month combinations
- Test with production-like volumes
- Monitor partition metrics post-deployment

---

### Q7: Explain indexing in Cosmos DB and how to optimize it.

**Answer:**

**How It Works:**
Cosmos DB automatically indexes all properties by default using inverted indexes.

**Components:**

**1. Indexing Mode:**
- `Consistent`: Index updated synchronously (default)
- `None`: No indexing (only point reads)

**2. Path Inclusion/Exclusion:**

**Optimized Example (E-commerce):**
```json
{
    "indexingMode": "consistent",
    "includedPaths": [
        {"path": "/category/?"},     // Frequently filtered
        {"path": "/price/?"},        // Used in ORDER BY
        {"path": "/inStock/?"}       // Boolean filter
    ],
    "excludedPaths": [
        {"path": "/*"},              // Exclude by default
        {"path": "/description/?"},  // Large text field
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

**3. Composite Indexes:**
Required for multi-property ORDER BY:
```sql
-- Requires composite index
SELECT * FROM c 
WHERE c.category = "electronics"
ORDER BY c.price DESC
```

**Optimization Benefits:**
- **Write Cost**: Reduced 40-50% by excluding unnecessary properties
- **Storage**: Smaller index footprint
- **Query Performance**: Composite indexes enable efficient sorts

**When to Exclude:**
- Large text fields (descriptions, comments)
- Binary data
- Properties never queried
- High-cardinality arrays

---

### Q8: How do transactions work in Cosmos DB?

**Answer:**

**ACID Transactions Within Single Partition:**

**Implementation Methods:**

**1. Transactional Batch:**
```csharp
var batch = container.CreateTransactionalBatch(new PartitionKey("user123"))
    .CreateItem(order)
    .CreateItem(orderItem1)
    .CreateItem(orderItem2)
    .UpdateItem("inventory1", inventory);

var response = await batch.ExecuteAsync();
// All-or-nothing execution
```

**Constraints:**
- Same partition key required
- Max 100 operations
- Max 2MB size
- No cross-partition support

**2. Stored Procedures:**
```javascript
function transferMoney(fromAccount, toAccount, amount) {
    // Read both accounts
    // Validate balance
    // Update both
    // All atomic within same partition
}
```

**Benefits:**
- Custom validation logic
- Complex multi-step operations
- Reduced network round trips
- Server-side execution

**Key Point:**
Design partition key to group transactionally related data together. Cannot do cross-partition transactions.

**Example Use Cases:**
- Order + order items (same userId)
- Account transfers (same accountId)
- Inventory updates (same productId)

---

### Q9: What are the different provisioning models and when to use each?

**Answer:**

**1. Manual (Provisioned)**

**When:**
- Predictable, steady workloads
- >60% sustained utilization
- Production with known traffic patterns

**Cost:** Fixed hourly
```
10K RU/s = ~$0.58/hour = $423/month
```

**2. Autoscale**

**When:**
- Variable traffic patterns
- Unpredictable demand
- <60% average utilization
- New applications

**Cost:** Variable
```
10K max RU/s:
- At 100%: $0.87/hour
- At 40%: ~$0.35/hour
- Average: ~$255/month (40% savings)
```

**3. Serverless**

**When:**
- Intermittent workloads
- Dev/test environments
- Very low volume (<37 RU/s avg)
- Proof of concepts

**Cost:** Per request
```
$0.25 per million RUs
10M RUs/month = $2.50
```

**Limits:**
- Max 5K RU/s per partition
- Max 50 GB storage
- No multi-region writes

**Decision Matrix:**
```
Known steady traffic → Manual
Variable traffic → Autoscale  
Low/intermittent → Serverless
New app (unknown) → Autoscale
```

---

### Q10: How do you handle throttling (429 errors)?

**Answer:**

**Understanding 429:**
Occurs when RU consumption exceeds provisioned RU/s. Response includes `RetryAfter` header.

**Handling Strategies:**

**1. SDK Automatic Retry:**
```csharp
var options = new CosmosClientOptions
{
    MaxRetryAttemptsOnRateLimitedRequests = 9,
    MaxRetryWaitTimeOnRateLimitedRequests = TimeSpan.FromSeconds(30)
};
```

**2. Exponential Backoff:**
```csharp
int retries = 0;
while (retries < 5)
{
    try
    {
        var response = await container.CreateItemAsync(item);
        break;
    }
    catch (CosmosException ex) when (ex.StatusCode == HttpStatusCode.TooManyRequests)
    {
        var delay = ex.RetryAfter ?? TimeSpan.FromSeconds(Math.Pow(2, retries));
        await Task.Delay(delay);
        retries++;
    }
}
```

**3. Proactive Solutions:**
- Monitor normalized RU consumption
- Set alerts at 70-80% utilization
- Scale before hitting limits
- Use autoscale for variable workloads
- Optimize queries
- Implement caching

**4. Architecture Solutions:**
- Queue-based processing for batches
- Change Feed for async processing
- Distribute load across time

---

## Advanced Level

### Q11: Explain Cosmos DB's global distribution architecture.

**Answer:**

**Architecture Hierarchy:**

```
Cosmos DB Account (Global)
├── Region 1 (Primary Write)
│   ├── Partition 1: [4 replicas across fault domains]
│   └── Partition 2: [4 replicas]
├── Region 2 (Read/Write)
│   ├── Partition 1: [4 replicas]
│   └── Partition 2: [4 replicas]
└── Region N
```

**Physical Architecture:**

**1. Replica Sets:**
- Each partition has 4 replicas
- Distributed across fault/update domains
- Majority quorum for writes (3/4)

**2. Replication:**

**Single-Region Write:**
```
Client → Primary Region
Primary → Write to 3/4 replicas (quorum)
Primary → Async replication to all secondaries
Secondaries → Update their 4 replicas
```

**Multi-Region Write:**
```
Client → Nearest Region
Region → Write locally (3/4 quorum)
Region → Async replication to all other regions
Conflict Resolution → LWW or custom
```

**3. Consistency Implementation:**

- **Strong**: Synchronous replication to all regions (cross-region quorum)
- **Bounded Staleness**: Lag bounded by K versions or T time
- **Session**: Client session token tracks causality
- **Eventual**: Async replication, no coordination

**4. Failover:**

**Automatic:**
- Health monitoring detects failures
- Redirects to secondary (based on priority)
- Time: <1 hour typically

**Manual:**
- Admin changes region priority
- Immediate (seconds to minutes)
- Zero data loss with appropriate consistency

---

### Q12: Design a multi-tenant SaaS application using Cosmos DB.

**Answer:**

**Isolation Strategies:**

**1. Database-per-Tenant (Highest Isolation)**
```
Account
├── Database: tenant-1
├── Database: tenant-2
└── Database: tenant-N
```

**Pros:** Strong isolation, easy backup/restore
**Cons:** Management overhead, higher costs

**2. Container-per-Tenant**
```
Database: myapp
├── Container: tenant-1-orders
├── Container: tenant-2-orders
```

**Pros:** Good isolation, easier management
**Cons:** Still costly, container limits

**3. Partition-Key-per-Tenant (Recommended)**
```javascript
{
    "id": "order123",
    "tenantId": "tenant-1",  // Partition key
    "userId": "user456",
    "data": {...}
}
```

**Pros:** Cost-efficient, scales well
**Cons:** No storage isolation, noisy neighbor risk

**Enhanced with Composite:**
```javascript
{
    "id": "order123",
    "partitionKey": "tenant-1-user456",
    "tenantId": "tenant-1",
    "userId": "user456"
}
```

**Security Implementation:**
```csharp
// Resource tokens for tenant isolation
public async Task<string> GetTenantToken(string tenantId)
{
    var permission = new Permission
    {
        Id = $"tenant-{tenantId}",
        PermissionMode = PermissionMode.All,
        ResourcePartitionKey = new PartitionKey(tenantId)
    };
    
    var response = await user.UpsertPermissionAsync(permission);
    return response.Resource.Token;
}

// Client uses tenant-specific token
var client = new CosmosClient(endpoint, tenantToken);
```

**Billing & Metering:**
```csharp
// Track per-tenant consumption
var query = container.GetItemQueryIterator<Usage>(
    $"SELECT SUM(c._ru) as totalRU FROM c WHERE c.tenantId = '{tenantId}'",
    requestOptions: new QueryRequestOptions
    {
        PartitionKey = new PartitionKey(tenantId)
    }
);
```

**Recommendation:**
Partition-key-per-tenant for most SaaS applications, with resource tokens for security.

---

### Q13: How do you implement a real-time analytics pipeline?

**Answer:**

**Architecture:**
```
Cosmos DB (OLTP) → Change Feed → Processing → Analytics Store
                                     ↓
                          Azure Functions/Stream Analytics
                                     ↓
                        Event Hub → Downstream Systems
                                     ↓
                              Synapse/Power BI
```

**Implementation:**

**1. Change Feed Processor:**
```csharp
var processor = container
    .GetChangeFeedProcessorBuilder<Order>("analytics", HandleChanges)
    .WithInstanceName("worker-1")
    .WithLeaseContainer(leaseContainer)
    .WithPollInterval(TimeSpan.FromSeconds(1))
    .Build();

async Task HandleChanges(
    ChangeFeedProcessorContext context,
    IReadOnlyCollection<Order> changes,
    CancellationToken cancellationToken)
{
    foreach (var order in changes)
    {
        // Real-time aggregations
        await UpdateMetrics(order);
        
        // Send to Event Hub
        await eventHub.SendAsync(new EventData(
            Encoding.UTF8.GetBytes(JsonSerializer.Serialize(order))
        ));
    }
}

await processor.StartAsync();
```

**2. Materialized Views:**
```csharp
async Task UpdateMetrics(Order order)
{
    var date = order.OrderDate.ToString("yyyy-MM-dd");
    var metricId = $"revenue-{date}";
    
    try
    {
        var metrics = await metricsContainer.ReadItemAsync<Metrics>(
            metricId, new PartitionKey(date));
        
        metrics.Resource.TotalRevenue += order.Total;
        metrics.Resource.OrderCount++;
        
        await metricsContainer.ReplaceItemAsync(
            metrics.Resource, metricId, new PartitionKey(date));
    }
    catch (CosmosException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
    {
        await metricsContainer.CreateItemAsync(new Metrics
        {
            Id = metricId,
            Date = date,
            TotalRevenue = order.Total,
            OrderCount = 1
        });
    }
}
```

**3. Synapse Link:**
```sql
-- Analytical queries without impacting OLTP
SELECT 
    category,
    COUNT(*) as orderCount,
    SUM(total) as revenue
FROM OPENROWSET(
    'CosmosDB',
    'Account=myaccount;Database=db;Key=...',
    orders
) WITH (category VARCHAR(50), total DECIMAL(10,2))
WHERE orderDate >= DATEADD(day, -7, GETDATE())
GROUP BY category
```

**4. Stream Analytics:**
```sql
-- Real-time aggregation
SELECT
    System.Timestamp() AS WindowEnd,
    category,
    COUNT(*) AS OrderCount,
    SUM(total) AS Revenue
INTO [powerbi-output]
FROM [cosmosdb-input]
TIMESTAMP BY orderDate
GROUP BY category, TumblingWindow(Duration(minute, 5))
```

**Benefits:**
- Real-time (sub-second latency)
- No OLTP impact (Change Feed)
- Scalable (parallel processing)
- Flexible (multiple consumers)

---

### Q14: How do you optimize a container with hot partition issues?

**Answer:**

**Diagnosis:**

**1. Identify Hot Partitions:**
```kusto
AzureDiagnostics
| where Category == "PartitionKeyRUConsumption"
| summarize TotalRU = sum(todouble(requestCharge_s)) 
  by partitionKeyRangeId_s
| top 10 by TotalRU desc
```

**2. Check Distribution:**
```kusto
AzureDiagnostics
| where Category == "PartitionKeyStatistics"
| summarize StorageKB = sum(todouble(sizeKb_s)) by partitionKey_s
| order by StorageKB desc
```

**Solutions:**

**1. Redesign Partition Key:**

**Before (Hot Partition):**
```javascript
{
    "id": "order123",
    "partitionKey": "productId",  // Popular products get hot
    "productId": "iphone15"
}
```

**After (Better Distribution):**
```javascript
{
    "id": "order123",
    "partitionKey": "userId-2024-01",  // User + time
    "productId": "iphone15",
    "userId": "user456"
}
```

**2. Synthetic Partition Key:**
```javascript
{
    "id": "order123",
    "partitionKey": hash(orderId) % 100,  // Distribute across 100 partitions
    "originalKey": "productId"
}
```

**3. Add Randomness:**
```javascript
{
    "id": "order123",
    "partitionKey": `${productId}-${random(1, 10)}`,  // Suffix 1-10
    "productId": "iphone15"
}
```

**4. Time-Based Partitioning:**
```javascript
{
    "id": "order123",
    "partitionKey": `${date}-${orderId}`,
    "createdDate": "2024-01-04"
}
```

**5. Increase RU/s (Temporary):**
- Scale up to handle load while redesigning
- Monitor if helps or if redesign needed

**Migration Strategy:**
1. Create new container with optimized key
2. Use Change Feed to migrate data
3. Transform partition key during migration
4. Switch application to new container
5. Validate and decommission old container

**Prevention:**
- Design with high cardinality from start
- Test with production-like data
- Monitor partition metrics
- Regular reviews of access patterns

---

### Q15: Explain cost optimization strategies for Cosmos DB.

**Answer:**

**1. RU/s Optimization:**

**Right-Sizing:**
```
Monitor: Normalized RU Consumption
< 30%: Reduce or use autoscale
30-60%: Consider autoscale
60-80%: Good balance
> 80%: Scale up or optimize
```

**Autoscale vs Manual:**
```
Manual 20K: $1,168/month
Autoscale 20K (40% avg): $420/month
Savings: $748/month (64%)
```

**Time-Based Scaling:**
```csharp
// Business hours: 20K, Off-hours: 5K
Weighted avg: 11,250 RU/s vs 20K constant
Savings: 44%
```

**2. Query Optimization:**

**Point Reads:**
```
Query: 5 RUs
Point Read: 1 RU
Savings: 80%
```

**Specific Properties:**
```sql
SELECT * FROM c            -- 10 RUs
SELECT c.id, c.name FROM c -- 6 RUs
Savings: 40%
```

**3. Storage Optimization:**

**TTL:**
```
Before: 500GB × $0.25 = $125/month
After: 50GB × $0.25 = $12.50/month
Savings: $112.50/month
```

**Index Optimization:**
```json
{
    "excludedPaths": ["/largeText/?"]
}
Savings: 40% on writes, 30% on storage
```

**4. Caching:**
```
80% cache hit rate
1M reads → 200K Cosmos reads
Savings: 800K RUs/day
```

**5. Multi-Region:**
```
Minimize regions
1 region: $584/month
3 regions: $1,752/month
Remove 1 unused: Save $584/month
```

**6. Serverless for Dev/Test:**
```
Dev/Test Manual 400 RU/s: $23/month
Serverless (low usage): $2-5/month
Savings: $18-21/month per environment
```

**Total Potential Savings:**
```
RU optimization: 40-60%
Query optimization: 20-40%
Storage optimization: 30-50%
Combined: 50-70% reduction possible
```

---

