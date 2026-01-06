# Azure Cosmos DB - Part 4: Security, Backup & Monitoring

## Table of Contents
1. [Security](#security)
2. [Backup & Restore](#backup--restore)
3. [Diagnostics & Metrics](#diagnostics--metrics)

---

## 1. Security

### Authentication Methods

**1. Primary/Secondary Keys (Master Keys)**

**When to Use:**
- Internal backend services
- Administrative operations
- Development/testing

**Usage:**
```csharp
var client = new CosmosClient(
    accountEndpoint: "https://myaccount.documents.azure.com:443/",
    authKeyOrResourceToken: "your-primary-key-here"
);
```

**Best Practices:**
- 🔒 Never expose in client code
- 🔒 Store in Azure Key Vault
- 🔒 Rotate keys regularly
- 🔒 Use read-only keys for read operations

**2. Resource Tokens**

**When to Use:**
- Mobile applications
- Web clients
- Limited-scope access
- User-specific data access

**Creating Resource Token:**
```csharp
// Define permission
var permission = new Permission
{
    Id = $"user-{userId}-permission",
    PermissionMode = PermissionMode.Read,  // or All
    ResourceUri = $"dbs/mydb/colls/orders/docs/{userId}"
};

// Create permission
var response = await user.CreatePermissionAsync(permission);
string resourceToken = response.Resource.Token;
```

**3. Azure Active Directory (AAD)**

**When to Use:**
- Azure-hosted applications
- Services using Managed Identities
- Enterprise applications
- Zero-trust security

**Managed Identity:**
```csharp
var credential = new DefaultAzureCredential();
var client = new CosmosClient(
    accountEndpoint: "https://myaccount.documents.azure.com:443/",
    tokenCredential: credential
);
```

### Role-Based Access Control (RBAC)

**Create Custom Role:**
```bash
az cosmosdb sql role definition create \
  --account-name myCosmosAccount \
  --resource-group myResourceGroup \
  --body '{
    "RoleName": "Read-Only Data Access",
    "Type": "CustomRole",
    "AssignableScopes": ["/dbs/mydb"],
    "Permissions": [{
      "DataActions": [
        "Microsoft.DocumentDB/databaseAccounts/readMetadata",
        "Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/read",
        "Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/executeQuery"
      ]
    }]
  }'
```

**Available Data Actions:**
- `readMetadata`
- `items/read`
- `items/create`
- `items/replace`
- `items/upsert`
- `items/delete`
- `executeQuery`
- `executeStoredProcedure`

### Network Security

**1. Firewall Rules**

```bash
# Allow specific IP
az cosmosdb update \
  --name myCosmosAccount \
  --resource-group myResourceGroup \
  --ip-range-filter "40.76.54.131,52.176.0.0/16"
```

**2. Virtual Network Service Endpoints**

```bash
# Enable service endpoint
az network vnet subnet update \
  --resource-group myResourceGroup \
  --vnet-name myVNet \
  --name mySubnet \
  --service-endpoints Microsoft.AzureCosmosDB

# Configure Cosmos DB
az cosmosdb update \
  --name myCosmosAccount \
  --resource-group myResourceGroup \
  --enable-virtual-network true \
  --virtual-network-rules "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/{vnet}/subnets/{subnet}"
```

**3. Private Endpoints**

**When to Use:**
- High security requirements
- Compliance needs (PCI-DSS, HIPAA)
- Isolated network architectures

```bash
az network private-endpoint create \
  --name myPrivateEndpoint \
  --resource-group myResourceGroup \
  --vnet-name myVNet \
  --subnet mySubnet \
  --private-connection-resource-id "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.DocumentDB/databaseAccounts/myCosmosAccount" \
  --group-id Sql \
  --connection-name myConnection
```

### Encryption

**1. Encryption at Rest (Automatic)**
- All data encrypted by default
- 256-bit AES encryption
- Microsoft-managed keys by default

**Customer-Managed Keys:**
```bash
az cosmosdb update \
  --name myCosmosAccount \
  --resource-group myResourceGroup \
  --key-vault-key-uri "https://myvault.vault.azure.net/keys/mykey"
```

**2. Encryption in Transit**
- TLS 1.2+ for all connections
- Enforced by default
- Cannot be disabled

### Audit Logging

**Enable Diagnostics:**
```bash
az monitor diagnostic-settings create \
  --name cosmosdbdiagnostics \
  --resource /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.DocumentDB/databaseAccounts/myCosmosAccount \
  --logs '[
    {
      "category": "DataPlaneRequests",
      "enabled": true
    },
    {
      "category": "QueryRuntimeStatistics",
      "enabled": true
    }
  ]' \
  --workspace /subscriptions/{sub}/resourcegroups/{rg}/providers/microsoft.operationalinsights/workspaces/myworkspace
```

**Query Logs:**
```kusto
// Failed authentication attempts
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.DOCUMENTDB"
| where Category == "DataPlaneRequests"
| where statusCode_s >= 400
| project TimeGenerated, activityId_g, statusCode_s, clientIpAddress_s
```

---

## 2. Backup & Restore

### Backup Modes

**1. Periodic Backup (Default)**

**Configuration:**
- **Backup Interval**: 1-4 hours (default: 4 hours)
- **Backup Retention**: 8 hours to 30 days (default: 8 hours)
- **Redundancy**: Geo-redundant, zone-redundant, or locally-redundant

```bash
az cosmosdb update \
  --name myCosmosAccount \
  --resource-group myResourceGroup \
  --backup-interval 60 \
  --backup-retention 24
```

**Limitations:**
- Cannot restore to point in time
- Restore requires Microsoft support ticket
- Restoration time: several hours

**2. Continuous Backup (Point-in-Time Restore)**

**Features:**
- Continuous backup of changes
- Restore to any point in last 7-30 days
- Self-service restore
- Granular restore (account/database/container)

**Enable:**
```bash
az cosmosdb create \
  --name myCosmosAccount \
  --resource-group myResourceGroup \
  --backup-policy-type Continuous \
  --continuous-tier Continuous30Days
```

**Point-in-Time Restore:**
```bash
# Restore specific container
az cosmosdb sql container restore \
  --account-name myCosmosAccount \
  --resource-group myResourceGroup \
  --database-name myDatabase \
  --name myContainer \
  --restore-timestamp "2024-01-04T10:30:00Z"
```

### Comparison: Periodic vs Continuous

| Feature | Periodic | Continuous |
|---------|----------|------------|
| **Backup Frequency** | 1-4 hours | Continuous |
| **Retention** | 8 hours - 30 days | 7 or 30 days |
| **Restore Granularity** | Latest backup | Any point in time |
| **Restore Method** | Support ticket | Self-service |
| **Restore Time** | Hours | Minutes |
| **Cost** | Included | +15-20% |

### Global Failover

**Automatic Failover:**

```bash
az cosmosdb update \
  --name myCosmosAccount \
  --resource-group myResourceGroup \
  --enable-automatic-failover true \
  --locations regionName=eastus failoverPriority=0 \
  --locations regionName=westus failoverPriority=1
```

**Manual Failover:**
```bash
az cosmosdb failover-priority-change \
  --name myCosmosAccount \
  --resource-group myResourceGroup \
  --failover-policies eastus=1 westus=0
```

**Failover SLAs:**
- **Automatic**: < 1 hour
- **Manual**: Seconds to minutes
- **Data Loss**: Depends on consistency level

---

## 3. Diagnostics & Metrics

### Key Metrics

**1. Request Metrics**

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| Total Requests | All requests | Baseline + 3σ |
| Successful Requests | HTTP 200-299 | < 95% |
| Throttled Requests (429) | Rate-limited | > 1% |
| Server Errors (5xx) | Server failures | > 0.1% |

**2. Latency Metrics**

| Metric | Description | Target |
|--------|-------------|--------|
| Server-Side Latency | Cosmos processing time | < 10ms (P99) |
| End-to-End Latency | Total request time | < 50ms (P99) |

**3. Throughput Metrics**

| Metric | Description | Alert |
|--------|-------------|-------|
| Normalized RU Consumption | % of provisioned RUs | > 80% |
| Total Request Units | RUs consumed | Track trends |
| Max RUs Per Second | Peak consumption | Monitor |

**4. Storage Metrics**

| Metric | Description | Alert |
|--------|-------------|-------|
| Data Usage | Total data storage | 80% of limit |
| Index Usage | Index storage | Monitor cost |
| Document Count | Total documents | Track growth |

### Diagnostic Logs

**Log Categories:**

1. **DataPlaneRequests**: All data operations, RU charges, latency
2. **QueryRuntimeStatistics**: Query execution details
3. **PartitionKeyStatistics**: Storage per partition
4. **PartitionKeyRUConsumption**: RU consumption per partition
5. **ControlPlaneRequests**: Account management operations

### Kusto Queries

**Top Queries by RU:**
```kusto
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.DOCUMENTDB"
| where Category == "DataPlaneRequests"
| summarize TotalRU = sum(todouble(requestCharge_s)), 
            Count = count() 
  by queryText_s
| top 10 by TotalRU desc
```

**Throttling Analysis:**
```kusto
AzureDiagnostics
| where Category == "DataPlaneRequests"
| where statusCode_s == "429"
| summarize ThrottleCount = count() by 
    bin(TimeGenerated, 5m), 
    collectionName_s
| render timechart
```

**Slow Queries:**
```kusto
AzureDiagnostics
| where Category == "DataPlaneRequests"
| where todouble(duration_s) > 100
| project TimeGenerated, 
          queryText_s, 
          duration_s, 
          requestCharge_s
| order by duration_s desc
```

**Hot Partitions:**
```kusto
AzureDiagnostics
| where Category == "PartitionKeyRUConsumption"
| summarize TotalRU = sum(todouble(requestCharge_s)) by 
    partitionKeyRangeId_s,
    collectionName_s
| top 10 by TotalRU desc
```

### Alerts

**High RU Consumption:**
```bash
az monitor metrics alert create \
  --name "High RU Consumption" \
  --resource-group myResourceGroup \
  --scopes /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.DocumentDB/databaseAccounts/myaccount \
  --condition "max Normalized RU Consumption > 80" \
  --window-size 5m \
  --evaluation-frequency 1m
```

**Throttling:**
```bash
az monitor metrics alert create \
  --name "Throttling Detected" \
  --resource-group myResourceGroup \
  --scopes /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.DocumentDB/databaseAccounts/myaccount \
  --condition "count Total Requests where StatusCode == 429 > 10" \
  --window-size 5m
```

### Client-Side Diagnostics

```csharp
var clientOptions = new CosmosClientOptions
{
    EnableContentResponseOnWrite = false,
    MaxRetryAttemptsOnRateLimitedRequests = 9,
    MaxRetryWaitTimeOnRateLimitedRequests = TimeSpan.FromSeconds(30)
};

var client = new CosmosClient(endpoint, key, clientOptions);

// Get diagnostics
ItemResponse<Product> response = await container.CreateItemAsync(product);
CosmosDiagnostics diagnostics = response.Diagnostics;

Console.WriteLine($"Latency: {diagnostics.GetClientElapsedTime()}");
Console.WriteLine($"RU charge: {response.RequestCharge}");
Console.WriteLine($"Diagnostics: {diagnostics}");
```

### Best Practices - Monitoring

- ✅ Set up alerts for key metrics (RU, throttling, latency)
- ✅ Monitor normalized RU consumption (target 60-80%)
- ✅ Track P95/P99 latency, not just average
- ✅ Create dashboards for real-time visibility
- ✅ Enable diagnostic logs for production
- ✅ Send logs to Log Analytics
- ✅ Create saved queries for investigations
- ✅ Review metrics weekly for trends
- ✅ Tune alert thresholds to reduce noise
- ✅ Test alert configurations
