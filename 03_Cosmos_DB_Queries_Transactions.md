# Azure Cosmos DB - Part 3: Queries, Transactions & Stored Procedures

## Table of Contents
1. [Query Model](#query-model)
2. [Transactions](#transactions)
3. [Stored Procedures, Triggers & UDFs](#stored-procedures-triggers--udfs)
4. [Change Feed](#change-feed)

---

## 1. Query Model

### Cosmos DB SQL Syntax

Cosmos DB uses a SQL-like syntax for querying JSON documents.

### Basic Queries

**SELECT:**
```sql
-- Select all documents
SELECT * FROM c

-- Select specific properties
SELECT c.name, c.price FROM c

-- With aliases
SELECT c.name AS productName, c.price AS cost FROM c
```

**WHERE Clause:**
```sql
-- Equality
SELECT * FROM c WHERE c.category = "electronics"

-- Comparison
SELECT * FROM c WHERE c.price > 100 AND c.price <= 500

-- IN operator
SELECT * FROM c WHERE c.status IN ("active", "pending")

-- String functions
SELECT * FROM c WHERE STARTSWITH(c.name, "Pro")
SELECT * FROM c WHERE CONTAINS(c.description, "wireless")

-- NULL checks
SELECT * FROM c WHERE IS_DEFINED(c.discount)
```

**ORDER BY:**
```sql
-- Ascending
SELECT * FROM c ORDER BY c.price

-- Descending
SELECT * FROM c ORDER BY c.price DESC

-- Multiple properties (requires composite index)
SELECT * FROM c ORDER BY c.category ASC, c.price DESC
```

**Pagination:**
```sql
-- Using OFFSET LIMIT
SELECT * FROM c ORDER BY c.timestamp DESC
OFFSET 0 LIMIT 10  -- First page

SELECT * FROM c ORDER BY c.timestamp DESC
OFFSET 10 LIMIT 10  -- Second page
```

### Advanced Queries

**Joins (Within Document):**
```sql
-- Join with arrays in same document
SELECT p.name, t.tagName
FROM products p
JOIN t IN p.tags
```

**Aggregations:**
```sql
-- COUNT
SELECT VALUE COUNT(1) FROM c WHERE c.category = "books"

-- SUM
SELECT SUM(c.price) AS totalValue FROM c

-- AVG
SELECT AVG(c.rating) AS averageRating FROM c

-- MIN/MAX
SELECT MIN(c.price) AS cheapest, MAX(c.price) AS mostExpensive FROM c
```

**Array Functions:**
```sql
-- ARRAY_CONTAINS
SELECT * FROM c WHERE ARRAY_CONTAINS(c.tags, "electronics")

-- ARRAY_LENGTH
SELECT * FROM c WHERE ARRAY_LENGTH(c.reviews) > 10
```

**Spatial Queries:**
```sql
-- Distance calculation
SELECT c.name, ST_DISTANCE(c.location, {
    'type': 'Point',
    'coordinates': [31.9, 72.2]
}) AS distance
FROM c
WHERE ST_DISTANCE(c.location, {
    'type': 'Point',
    'coordinates': [31.9, 72.2]
}) < 30000  -- Within 30km
```

### Query Execution (.NET SDK)

**Basic Query:**
```csharp
var queryDefinition = new QueryDefinition(
    "SELECT * FROM c WHERE c.category = @category"
)
.WithParameter("@category", "electronics");

var iterator = container.GetItemQueryIterator<Product>(queryDefinition);

var results = new List<Product>();
while (iterator.HasMoreResults)
{
    var response = await iterator.ReadNextAsync();
    results.AddRange(response);
    
    Console.WriteLine($"RU consumed: {response.RequestCharge}");
}
```

**With Partition Key (Most Efficient):**
```csharp
var queryDefinition = new QueryDefinition(
    "SELECT * FROM c WHERE c.price < @price"
)
.WithParameter("@price", 1000);

var iterator = container.GetItemQueryIterator<Product>(
    queryDefinition,
    requestOptions: new QueryRequestOptions
    {
        PartitionKey = new PartitionKey("electronics"),
        MaxItemCount = 100  // Page size
    }
);
```

**Pagination:**
```csharp
string continuationToken = null;

do
{
    var iterator = container.GetItemQueryIterator<Product>(
        queryDefinition,
        continuationToken: continuationToken,
        requestOptions: new QueryRequestOptions
        {
            MaxItemCount = 10
        }
    );
    
    var response = await iterator.ReadNextAsync();
    
    foreach (var item in response)
    {
        // Process item
    }
    
    continuationToken = response.ContinuationToken;
    
} while (continuationToken != null);
```

---

## 2. Transactions

### What are Transactions in Cosmos DB?

- ACID transactions within a single logical partition
- All-or-nothing execution
- Implemented via Transactional Batch or Stored Procedures

**Limitations:**
- Must be within same partition key
- Max 100 operations per batch
- Max 2MB total size
- No cross-partition transactions

### Transactional Batch

**Syntax:**
```csharp
// All items must have same partition key
var partitionKey = new PartitionKey("user123");

var batch = container.CreateTransactionalBatch(partitionKey)
    .CreateItem(new Order { id = "order1", userId = "user123" })
    .UpsertItem(new Order { id = "order2", userId = "user123" })
    .ReplaceItem("order3", new Order { id = "order3", userId = "user123" })
    .DeleteItem("order4");

// Execute batch - either all succeed or all fail
TransactionalBatchResponse response = await batch.ExecuteAsync();

if (response.IsSuccessStatusCode)
{
    Console.WriteLine("All operations succeeded");
}
else
{
    Console.WriteLine($"Batch failed: {response.StatusCode}");
}
```

**Use Cases:**
- Creating order with multiple line items
- Updating related documents atomically
- Deleting parent and child documents together
- Account balance transfers within same account

**Example: Order Processing**
```csharp
var orderId = Guid.NewGuid().ToString();
var userId = "user123";

var batch = container.CreateTransactionalBatch(new PartitionKey(userId))
    .CreateItem(new Order
    {
        id = orderId,
        userId = userId,
        total = 100.00m,
        status = "pending"
    })
    .CreateItem(new OrderItem
    {
        id = Guid.NewGuid().ToString(),
        userId = userId,
        orderId = orderId,
        productId = "prod1",
        quantity = 2
    })
    .CreateItem(new OrderItem
    {
        id = Guid.NewGuid().ToString(),
        userId = userId,
        orderId = orderId,
        productId = "prod2",
        quantity = 1
    });

var response = await batch.ExecuteAsync();
```

---

## 3. Stored Procedures, Triggers & UDFs

### Stored Procedures

**What are Stored Procedures?**
- Server-side JavaScript code
- Execute transactions with custom logic
- Guaranteed ACID within partition
- Can return custom responses

**Basic Stored Procedure:**
```javascript
// Create stored procedure
function createDocument(doc) {
    var collection = getContext().getCollection();
    var collectionLink = collection.getSelfLink();
    
    var accepted = collection.createDocument(
        collectionLink,
        doc,
        function(err, documentCreated) {
            if (err) throw new Error('Error: ' + err.message);
            getContext().getResponse().setBody(documentCreated.id);
        }
    );
    
    if (!accepted) throw new Error('Document creation not accepted');
}
```

**Transaction Example - Transfer:**
```javascript
function transfer(fromAccountId, toAccountId, amount) {
    var collection = getContext().getCollection();
    var collectionLink = collection.getSelfLink();
    var response = getContext().getResponse();
    
    // Read source account
    var fromQuery = `SELECT * FROM c WHERE c.id = "${fromAccountId}"`;
    var accepted = collection.queryDocuments(
        collectionLink,
        fromQuery,
        function(err, documents) {
            if (err) throw err;
            if (documents.length === 0) throw new Error('Source account not found');
            
            var fromAccount = documents[0];
            
            // Check balance
            if (fromAccount.balance < amount) {
                throw new Error('Insufficient funds');
            }
            
            // Deduct from source
            fromAccount.balance -= amount;
            
            collection.replaceDocument(
                fromAccount._self,
                fromAccount,
                function(err, updatedFrom) {
                    if (err) throw err;
                    
                    // Read destination
                    var toQuery = `SELECT * FROM c WHERE c.id = "${toAccountId}"`;
                    collection.queryDocuments(
                        collectionLink,
                        toQuery,
                        function(err, documents) {
                            if (err) throw err;
                            if (documents.length === 0) throw new Error('Destination not found');
                            
                            var toAccount = documents[0];
                            toAccount.balance += amount;
                            
                            collection.replaceDocument(
                                toAccount._self,
                                toAccount,
                                function(err, updatedTo) {
                                    if (err) throw err;
                                    response.setBody({
                                        success: true,
                                        fromBalance: updatedFrom.balance,
                                        toBalance: updatedTo.balance
                                    });
                                }
                            );
                        }
                    );
                }
            );
        }
    );
    
    if (!accepted) throw new Error('Query not accepted');
}
```

**Executing from .NET:**
```csharp
// Register stored procedure
Scripts scripts = container.Scripts;
await scripts.CreateStoredProcedureAsync(new StoredProcedureProperties
{
    Id = "transfer",
    Body = File.ReadAllText("transfer.js")
});

// Execute stored procedure
var response = await scripts.ExecuteStoredProcedureAsync<dynamic>(
    "transfer",
    new PartitionKey("account123"),
    new[] { "account123", "account456", 100.00 }
);
```

### Triggers

**Pre-Triggers:**
Execute before operation (validation, modification):

```javascript
function validateDocument() {
    var request = getContext().getRequest();
    var doc = request.getBody();
    
    // Validation
    if (!doc.name || doc.name.length < 3) {
        throw new Error('Name must be at least 3 characters');
    }
    
    // Auto-populate
    doc.createdDate = new Date().toISOString();
    
    request.setBody(doc);
}
```

**Post-Triggers:**
Execute after operation (logging, auditing):

```javascript
function logDocument() {
    var request = getContext().getRequest();
    var response = getContext().getResponse();
    var collection = getContext().getCollection();
    
    var createdDoc = response.getBody();
    
    // Create audit log
    var auditLog = {
        id: generateGUID(),
        documentId: createdDoc.id,
        operation: 'CREATE',
        timestamp: new Date().toISOString()
    };
    
    collection.createDocument(
        collection.getSelfLink(),
        auditLog
    );
}
```

**Using Triggers:**
```csharp
await container.CreateItemAsync(
    product,
    requestOptions: new ItemRequestOptions
    {
        PreTriggers = new[] { "validateDocument" },
        PostTriggers = new[] { "logDocument" }
    }
);
```

### User-Defined Functions (UDFs)

**Creating UDF:**
```javascript
function calculateTax(price) {
    return price * 0.2;  // 20% tax
}
```

**Register UDF:**
```csharp
await container.Scripts.CreateUserDefinedFunctionAsync(
    new UserDefinedFunctionProperties
    {
        Id = "calculateTax",
        Body = @"function calculateTax(price) { return price * 0.2; }"
    }
);
```

**Use in Query:**
```sql
SELECT 
    c.name,
    c.price,
    udf.calculateTax(c.price) AS tax,
    c.price + udf.calculateTax(c.price) AS total
FROM c
```

---

## 4. Change Feed

### What is Change Feed?

- Persistent record of changes to container
- Ordered by modification time per partition
- Can process changes in real-time
- Supports multiple consumers

**Use Cases:**
- Event-driven architectures
- Real-time analytics
- Data synchronization
- Materialized views
- Notifications

### Change Feed Processor (.NET)

```csharp
Container leaseContainer = client.GetContainer("db", "leases");
Container monitoredContainer = client.GetContainer("db", "products");

ChangeFeedProcessor processor = monitoredContainer
    .GetChangeFeedProcessorBuilder<Product>("productProcessor", HandleChangesAsync)
    .WithInstanceName("consoleApp")
    .WithLeaseContainer(leaseContainer)
    .WithStartTime(DateTime.MinValue.ToUniversalTime())
    .Build();

async Task HandleChangesAsync(
    ChangeFeedProcessorContext context,
    IReadOnlyCollection<Product> changes,
    CancellationToken cancellationToken)
{
    foreach (var product in changes)
    {
        // Process change
        Console.WriteLine($"Product {product.id} changed");
        
        // Update search index, send notification, etc.
        await UpdateSearchIndex(product);
    }
}

await processor.StartAsync();
```

### Best Practices

**Queries:**
- ✅ Always specify partition key when possible
- ✅ Use parameterized queries
- ✅ Limit results with TOP or OFFSET/LIMIT
- ✅ Create composite indexes for complex ORDER BY
- ✅ Use SELECT with specific properties, not SELECT *

**Transactions:**
- ✅ Keep transactions within same partition
- ✅ Keep batch operations under 100 items
- ✅ Implement retry logic for transient failures
- ✅ Use stored procedures for complex logic

**Stored Procedures:**
- ✅ Keep execution time under 5 seconds
- ✅ Check operation acceptance
- ✅ Handle errors with try-catch
- ✅ Return meaningful responses
- ✅ Test thoroughly before production
