# Azure Cosmos DB - Part 5: Cost Optimization & Best Practices

## Table of Contents
1. [Cost Optimization](#cost-optimization)
2. [Best Practices](#best-practices)

---

## 1. Cost Optimization

### Understanding Cosmos DB Costs

**Cost Components:**
1. **Throughput (RU/s)**: 65-75% of bill
2. **Storage**: Data + index storage
3. **Backup**: Continuous backup adds cost
4. **Egress**: Data transfer out
5. **Multi-region**: Multiplies throughput cost

**Formula:**
```
Total Cost = (Throughput × Regions) + Storage + Backup + Egress
```

### RU/s Optimization

**1. Right-Size Provisioned Throughput**

**Monitor Utilization:**
```kusto
AzureMetrics
| where MetricName == "Normalized RU Consumption"
| summarize AvgUtilization = avg(Maximum) by bin(TimeGenerated, 1h)
```

**Strategy:**
```
< 30% utilization → Reduce or switch to autoscale
30-60% → Consider autoscale
60-80% → Good balance
> 80% → Increase or optimize queries
```

**Example Savings:**
```
Before: 20K RU/s manual (30% utilization) = $1,168/month
After: 15K autoscale (avg 40%) = ~$420/month
Savings: $748/month (64%)
```

**2. Use Autoscale for Variable Workloads**

**Breakeven Analysis:**
```
Manual: $X/hour
Autoscale at max: $1.5X/hour
Breakeven: < 67% average utilization

Example:
Manual 10K RU/s: $0.58/hour
Autoscale 10K max at 40% avg: ~$0.35/hour
```

**3. Serverless for Low Volume**

**Cost Comparison:**
```
Serverless: $0.25 per million RUs
Manual 400 RU/s: ~$23/month

Breakeven: ~92 million RUs/month
If avg < 37 RU/s → Serverless cheaper
```

**4. Database-Level Throughput**

**Scenario:**
```
Container A: 5K peak, 1K avg
Container B: 3K peak, 500 avg
Container C: 4K peak, 800 avg
```

**Container-level:** 12K total = $700/month
**Database-level:** 6K shared = $350/month
**Savings:** $350/month (50%)

**5. Time-Based Scaling**

```csharp
// Azure Function - Scale based on time
[FunctionName("CosmosScaler")]
public static async Task Run(
    [TimerTrigger("0 0 8,18 * * *")] TimerInfo timer)
{
    var hour = DateTime.Now.Hour;
    int targetRU = (hour == 8) ? 20000 : 5000;
    await container.ReplaceThroughputAsync(targetRU);
}
```

**Savings:**
```
Business hours (10h): 20K RU/s
Off-hours (14h): 5K RU/s
Weighted avg: 11,250 RU/s vs 20K
Savings: 44%
```

### Query Optimization

**1. Point Reads vs Queries**

```csharp
// ❌ Query: ~5 RUs
var query = new QueryDefinition("SELECT * FROM c WHERE c.id = @id");

// ✅ Point Read: ~1 RU (5x cheaper!)
var response = await container.ReadItemAsync<Product>(
    id: "123",
    partitionKey: new PartitionKey("electronics")
);
```

**2. Select Specific Properties**

```sql
-- ❌ Full document: 10 RUs
SELECT * FROM c WHERE c.category = "electronics"

-- ✅ Specific properties: 6 RUs (40% savings)
SELECT c.id, c.name, c.price FROM c 
WHERE c.category = "electronics"
```

**3. Optimize Indexing**

```json
// Reduce write costs by 40%
{
    "includedPaths": [
        {"path": "/category/?"},
        {"path": "/price/?"}
    ],
    "excludedPaths": [
        {"path": "/*"}
    ]
}
```

**4. Implement Caching**

```csharp
// Application-level cache
private static MemoryCache _cache = new MemoryCache(new MemoryCacheOptions());

public async Task<Product> GetProductAsync(string id, string category)
{
    string cacheKey = $"{category}:{id}";
    
    if (_cache.TryGetValue(cacheKey, out Product cached))
        return cached;  // 0 RUs!
    
    var product = await container.ReadItemAsync<Product>(id, new PartitionKey(category));
    _cache.Set(cacheKey, product.Resource, TimeSpan.FromMinutes(5));
    
    return product.Resource;
}
```

**Impact:**
```
Without cache: 1M reads/day × 1 RU = 1M RUs/day
With 80% hit rate: 200K reads × 1 RU = 200K RUs/day
Savings: 800K RUs/day (~$6/month)
```

### Storage Optimization

**1. Implement TTL**

```csharp
var containerProperties = new ContainerProperties("logs", "/date")
{
    DefaultTimeToLive = 604800  // 7 days
};
```

**Cost Impact:**
```
Before: 500 GB × $0.25 = $125/month
After: 50 GB × $0.25 = $12.50/month
Savings: $112.50/month
```

**2. Compress Large Documents**

```csharp
public static string Compress(string text)
{
    byte[] buffer = Encoding.UTF8.GetBytes(text);
    using var ms = new MemoryStream();
    using (var gzip = new GZipStream(ms, CompressionMode.Compress))
    {
        gzip.Write(buffer, 0, buffer.Length);
    }
    return Convert.ToBase64String(ms.ToArray());
}

// Storage: 50 KB → 10 KB (80% reduction)
// RU cost: 10 RUs → 2 RUs (80% reduction)
```

**3. Optimize Index Storage**

```json
{
    "excludedPaths": [
        {"path": "/largeTextField/?"},
        {"path": "/metadata/*"}
    ]
}
```

**Savings:**
```
Before: Data 100GB + Index 100GB = 200GB
After: Data 100GB + Index 30GB = 130GB
Savings: $17.50/month
```

### Multi-Region Optimization

**Cost Multiplication:**
```
1 region: 10K RU/s = $584/month
2 regions: 10K × 2 = $1,168/month
3 regions: 10K × 3 = $1,752/month

Multi-region write: 25% premium
```

**Optimization:**
- Use only necessary regions
- Consider CDN for static content
- Remove underutilized regions
- Single-region writes when possible

### Cost Monitoring

**Daily RU Tracking:**
```kusto
AzureDiagnostics
| where Category == "DataPlaneRequests"
| summarize TotalRU = sum(todouble(requestCharge_s)) by bin(TimeGenerated, 1d)
| extend EstimatedCost = TotalRU / 1000000 * 0.25
```

**Container-Level Cost:**
```kusto
AzureDiagnostics
| where Category == "DataPlaneRequests"
| summarize TotalRU = sum(todouble(requestCharge_s)) by 
    collectionName_s,
    bin(TimeGenerated, 1d)
| extend DailyCost = TotalRU / 1000000 * 0.25
| summarize MonthlyCost = sum(DailyCost) * 30 by collectionName_s
```

### Cost Optimization Checklist

**Immediate (Quick Wins):**
- ✅ Review RU utilization - reduce if < 30%
- ✅ Switch to autoscale if variable traffic
- ✅ Implement TTL on temporary data
- ✅ Use point reads instead of queries
- ✅ Remove unused containers
- ✅ Optimize indexing policy

**Medium-Term (Weeks):**
- ✅ Implement caching strategy
- ✅ Optimize query patterns
- ✅ Database-level throughput
- ✅ Time-based scaling
- ✅ Reduce multi-region footprint

**Long-Term (Months):**
- ✅ Partition key optimization
- ✅ Archive to cheaper storage
- ✅ Migrate low-volume to serverless
- ✅ Data compression
- ✅ Regular cost audits

---

## 2. Best Practices

### Design Best Practices

**1. Partition Key Selection**
- Choose high cardinality (1000+ values)
- Ensure even distribution
- Align with query patterns
- Consider future growth
- Test with production volumes

**2. Data Modeling**
- Denormalize for read efficiency
- Embed related data accessed together
- Reference large/rarely accessed data
- Balance read vs write efficiency

**3. Container Design**
- Group by access patterns
- Avoid container per entity
- Use database-level throughput
- Plan for scale from day one

**4. Indexing Strategy**
- Index only queried properties
- Exclude large text/binaries
- Create composite indexes
- Review quarterly

### Application Development

**1. Connection Management**

```csharp
// ✅ Singleton - one client per application
private static readonly Lazy<CosmosClient> _client = new Lazy<CosmosClient>(() =>
{
    var options = new CosmosClientOptions
    {
        MaxRetryAttemptsOnRateLimitedRequests = 9,
        ConnectionMode = ConnectionMode.Direct,
        EnableContentResponseOnWrite = false
    };
    return new CosmosClient(endpoint, key, options);
});

// ❌ Don't create per request
```

**2. Error Handling**

```csharp
try
{
    var response = await container.CreateItemAsync(item);
}
catch (CosmosException ex) when (ex.StatusCode == HttpStatusCode.TooManyRequests)
{
    // 429 throttling - exponential backoff
    await Task.Delay(ex.RetryAfter ?? TimeSpan.FromSeconds(1));
}
catch (CosmosException ex) when (ex.StatusCode == HttpStatusCode.Conflict)
{
    // 409 conflict
}
catch (CosmosException ex)
{
    logger.LogError(ex, "Cosmos error: {StatusCode}", ex.StatusCode);
}
```

**3. Bulk Operations**

```csharp
// ✅ Bulk execution
var options = new CosmosClientOptions { AllowBulkExecution = true };

var tasks = new List<Task>();
foreach (var item in items)
{
    tasks.Add(container.CreateItemAsync(item));
}
await Task.WhenAll(tasks);

// ❌ Sequential
// foreach (var item in items)
//     await container.CreateItemAsync(item);  // SLOW!
```

**4. Pagination**

```csharp
// ✅ Continuation tokens
string continuationToken = null;
do
{
    var response = await iterator.ReadNextAsync();
    ProcessResults(response);
    continuationToken = response.ContinuationToken;
} while (continuationToken != null);

// ❌ Fetch all at once
```

**5. Optimistic Concurrency**

```csharp
// Use ETag
var response = await container.ReadItemAsync<Product>("123", new PartitionKey("electronics"));
var etag = response.ETag;

await container.ReplaceItemAsync(
    product,
    product.id,
    new PartitionKey(product.category),
    new ItemRequestOptions { IfMatchEtag = etag }
);
```

### Performance Best Practices

- ✅ Use Direct mode (default)
- ✅ Disable content on write
- ✅ Prefer async operations
- ✅ Batch related operations
- ✅ Include partition key in queries
- ✅ Use point reads when possible
- ✅ Implement caching
- ✅ Monitor query performance

### Security Best Practices

- ✅ Never hardcode connection strings
- ✅ Use Azure Key Vault
- ✅ Rotate keys regularly (90 days)
- ✅ Use Managed Identities
- ✅ Enable firewall rules
- ✅ Use Private Endpoints for sensitive data
- ✅ Implement least privilege
- ✅ Enable diagnostic logging

### Operational Best Practices

**1. Monitoring**
- Comprehensive alerting
- Monitor RU, latency, availability
- Track slow queries
- Regular performance reviews
- Establish baselines

**2. Backup & DR**
- Continuous backup for production
- Test restore quarterly
- Document DR runbooks
- Enable automatic failover
- Verify backup integrity

**3. Capacity Planning**
- Monitor growth trends
- Plan for seasonal variations
- Load test before releases
- Scale proactively
- Document procedures

**4. Cost Management**
- Review costs weekly
- Set budget alerts
- Optimize RU consumption
- Archive unused data
- Right-size throughput

### Common Pitfalls to Avoid

**❌ Don't:**
- Create client per request
- Ignore 429 errors
- Use SELECT * in queries
- Hard-code partition keys
- Over-provision throughput
- Index everything
- Use synchronous APIs
- Skip performance testing
- Use low-cardinality keys
- Expose master keys in clients

**✅ Do:**
- Singleton CosmosClient
- Implement retry logic
- Select specific properties
- Parameterize partition keys
- Start with autoscale
- Optimize indexing
- Use async/await
- Performance test
- High-cardinality keys
- Use resource tokens/AAD
