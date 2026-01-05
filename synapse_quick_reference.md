# Azure Synapse Analytics - Quick Reference Cheat Sheet

## Core Components at a Glance

| Component | Use Case | Key Feature |
|-----------|----------|-------------|
| **Dedicated SQL Pool** | Data warehouse | Predictable performance, provisioned |
| **Serverless SQL Pool** | Ad-hoc queries | Pay-per-query, no provisioning |
| **Spark Pools** | Big data processing | Distributed computing, notebooks |
| **Pipelines** | ETL/ELT | Data orchestration, 90+ connectors |
| **Data Explorer** | Real-time analytics | Streaming, KQL queries |

---

## Quick Start Commands

### Azure CLI - Create Workspace
```bash
az synapse workspace create \
  --name myworkspace \
  --resource-group myRG \
  --storage-account mystorageaccount \
  --file-system data \
  --sql-admin-login-user sqladmin \
  --sql-admin-login-password 'P@ssw0rd123' \
  --location eastus
```

### Create Spark Pool
```bash
az synapse spark pool create \
  --name sparkpool01 \
  --workspace-name myworkspace \
  --resource-group myRG \
  --node-count 3 \
  --node-size Small \
  --spark-version 3.3
```

---

## SQL Quick Reference

### Query Data Lake (Serverless)
```sql
-- Query CSV
SELECT * FROM OPENROWSET(
    BULK 'https://storage.dfs.core.windows.net/data/*.csv',
    FORMAT = 'CSV',
    PARSER_VERSION = '2.0',
    HEADER_ROW = TRUE
) AS data;

-- Query Parquet
SELECT * FROM OPENROWSET(
    BULK 'https://storage.dfs.core.windows.net/data/*.parquet',
    FORMAT = 'PARQUET'
) AS data;

-- Query JSON
SELECT * FROM OPENROWSET(
    BULK 'https://storage.dfs.core.windows.net/data/*.json',
    FORMAT = 'CSV',
    FIELDTERMINATOR = '0x0b',
    FIELDQUOTE = '0x0b'
) WITH (doc NVARCHAR(MAX)) AS data
CROSS APPLY OPENJSON(doc);
```

### Create External Table
```sql
-- 1. Create database
CREATE DATABASE MyDB;

-- 2. Create external data source
CREATE EXTERNAL DATA SOURCE MyDataSource
WITH (LOCATION = 'https://storage.dfs.core.windows.net/data');

-- 3. Create file format
CREATE EXTERNAL FILE FORMAT ParquetFormat
WITH (FORMAT_TYPE = PARQUET);

-- 4. Create external table
CREATE EXTERNAL TABLE MyTable (
    ID INT,
    Name NVARCHAR(100)
)
WITH (
    LOCATION = 'mytable/',
    DATA_SOURCE = MyDataSource,
    FILE_FORMAT = ParquetFormat
);
```

### Dedicated SQL Pool - Create Table
```sql
-- Hash distributed table
CREATE TABLE FactSales (
    SaleID INT NOT NULL,
    ProductID INT,
    Amount DECIMAL(10,2)
)
WITH (
    DISTRIBUTION = HASH(SaleID),
    CLUSTERED COLUMNSTORE INDEX
);

-- Replicated table (for dimensions)
CREATE TABLE DimProduct (
    ProductID INT NOT NULL,
    ProductName NVARCHAR(100)
)
WITH (
    DISTRIBUTION = REPLICATE,
    CLUSTERED COLUMNSTORE INDEX
);

-- Partitioned table
CREATE TABLE FactOrders (
    OrderID INT,
    OrderDate DATE
)
WITH (
    DISTRIBUTION = HASH(OrderID),
    PARTITION (OrderDate RANGE RIGHT 
        FOR VALUES ('2024-01-01', '2024-02-01'))
);
```

---

## PySpark Quick Reference

### Read Data
```python
# CSV
df = spark.read \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .csv("path/to/file.csv")

# Parquet
df = spark.read.parquet("path/to/file.parquet")

# Delta
df = spark.read.format("delta").load("path/to/delta")

# JSON
df = spark.read.json("path/to/file.json")

# From SQL Pool
df = spark.read \
    .format("com.microsoft.sqlserver.jdbc.spark") \
    .option("url", "jdbc:sqlserver://workspace.sql.azuresynapse.net") \
    .option("dbtable", "dbo.MyTable") \
    .load()
```

### Common Transformations
```python
from pyspark.sql.functions import *

# Select columns
df.select("col1", "col2")

# Filter
df.filter(col("amount") > 100)
df.where(col("status") == "active")

# Add column
df.withColumn("new_col", col("old_col") * 2)

# Rename column
df.withColumnRenamed("old_name", "new_name")

# Group by
df.groupBy("category").agg(
    count("*").alias("count"),
    sum("amount").alias("total"),
    avg("amount").alias("average")
)

# Join
df1.join(df2, df1.id == df2.id, "inner")

# Order
df.orderBy(col("amount").desc())

# Drop duplicates
df.dropDuplicates(["id"])

# Drop nulls
df.na.drop()
df.dropna(subset=["col1", "col2"])
```

### Write Data
```python
# Parquet
df.write.mode("overwrite").parquet("path/to/output")

# Delta
df.write.format("delta").mode("overwrite").save("path/to/delta")

# Partitioned
df.write.partitionBy("year", "month").parquet("path/to/output")

# To SQL Pool
df.write \
    .format("com.microsoft.sqlserver.jdbc.spark") \
    .option("url", "jdbc:sqlserver://workspace.sql.azuresynapse.net") \
    .option("dbtable", "dbo.OutputTable") \
    .mode("overwrite") \
    .save()
```

---

## Pipeline Quick Reference

### Copy Activity (JSON)
```json
{
    "name": "CopyActivity",
    "type": "Copy",
    "inputs": [{"referenceName": "SourceDataset"}],
    "outputs": [{"referenceName": "SinkDataset"}],
    "typeProperties": {
        "source": {
            "type": "DelimitedTextSource"
        },
        "sink": {
            "type": "ParquetSink"
        }
    }
}
```

### ForEach Activity
```json
{
    "name": "ForEachFile",
    "type": "ForEach",
    "typeProperties": {
        "items": {
            "value": "@pipeline().parameters.FileList",
            "type": "Expression"
        },
        "activities": [
            {
                "name": "CopyFile",
                "type": "Copy"
            }
        ]
    }
}
```

### Pipeline Expressions
```javascript
// Current date
@utcnow()

// Format date
@formatDateTime(utcnow(), 'yyyy-MM-dd')

// Pipeline parameters
@pipeline().parameters.ParamName

// Activity output
@activity('ActivityName').output.value

// Concatenate
@concat('prefix', pipeline().parameters.name, 'suffix')

// If condition
@if(equals(pipeline().parameters.env, 'prod'), 'Production', 'Development')
```

---

## Delta Lake Quick Reference

### Write Delta
```python
# Overwrite
df.write.format("delta").mode("overwrite").save("path")

# Append
df.write.format("delta").mode("append").save("path")

# With partition
df.write.format("delta") \
    .partitionBy("year", "month") \
    .save("path")
```

### Read Delta
```python
# Current version
df = spark.read.format("delta").load("path")

# Time travel (version)
df = spark.read.format("delta") \
    .option("versionAsOf", 0) \
    .load("path")

# Time travel (timestamp)
df = spark.read.format("delta") \
    .option("timestampAsOf", "2024-01-01") \
    .load("path")
```

### Delta Operations
```python
from delta.tables import DeltaTable

delta_table = DeltaTable.forPath(spark, "path")

# Update
delta_table.update(
    condition = "status = 'pending'",
    set = {"status": "'processed'"}
)

# Delete
delta_table.delete("date < '2024-01-01'")

# Merge (UPSERT)
delta_table.alias("target").merge(
    source_df.alias("source"),
    "target.id = source.id"
).whenMatchedUpdate(set = {
    "col1": "source.col1"
}).whenNotMatchedInsert(values = {
    "id": "source.id",
    "col1": "source.col1"
}).execute()

# Vacuum (remove old files)
delta_table.vacuum(168)  # hours

# History
delta_table.history().show()
```

---

## Performance Tuning Tips

### SQL Pool
- Use **hash distribution** for large tables (>2GB)
- Use **replicate distribution** for small dimensions (<2GB)
- Create **statistics** on filtered/joined columns
- Use **result set caching** for repeated queries
- Implement **materialized views** for aggregations
- **Partition** large tables by date

### Spark Pool
- Use **broadcast joins** for small tables (<10MB)
- **Cache** frequently used DataFrames
- **Repartition** data before expensive operations
- Use **columnar formats** (Parquet, Delta)
- **Persist** intermediate results
- **Coalesce** to reduce partition count after filters

### Data Lake
- Use **Parquet** for analytical workloads
- **Partition** data by frequently filtered columns
- Keep file sizes **between 128MB-1GB**
- Use **snappy compression**
- Organize by **date hierarchy** (year/month/day)

---

## Common Patterns

### Slowly Changing Dimension (Type 2)
```sql
-- Merge logic for SCD Type 2
MERGE INTO DimCustomer AS target
USING StagingCustomer AS source
ON target.CustomerID = source.CustomerID 
    AND target.IsActive = 1
WHEN MATCHED AND (
    target.Name != source.Name OR
    target.Email != source.Email
) THEN UPDATE SET
    IsActive = 0,
    EndDate = GETDATE()
WHEN NOT MATCHED THEN INSERT (
    CustomerID, Name, Email, 
    IsActive, StartDate
) VALUES (
    source.CustomerID, source.Name, source.Email,
    1, GETDATE()
);
```

### Incremental Load
```python
# Get max date from target
max_date = spark.sql("""
    SELECT MAX(LoadDate) as max_date 
    FROM target_table
""").collect()[0]['max_date']

# Load only new data
new_data = source_df.filter(
    col("LoadDate") > max_date
)

# Append to target
new_data.write.mode("append").saveAsTable("target_table")
```

### Error Handling in Pipeline
```json
{
    "activities": [{
        "name": "TryActivity",
        "type": "Copy",
        "dependsOn": [],
        "policy": {
            "retry": 3,
            "retryIntervalInSeconds": 30
        }
    }, {
        "name": "OnFailure",
        "type": "WebActivity",
        "dependsOn": [{
            "activity": "TryActivity",
            "dependencyConditions": ["Failed"]
        }]
    }]
}
```

---

## Monitoring Queries

### Pipeline Runs
```sql
SELECT 
    RunId,
    PipelineName,
    RunStart,
    RunEnd,
    Status,
    DATEDIFF(second, RunStart, RunEnd) AS DurationSeconds
FROM sys.dm_pipeline_runs
WHERE RunStart >= DATEADD(day, -7, GETDATE())
ORDER BY RunStart DESC;
```

### Spark Applications
```sql
SELECT 
    ApplicationId,
    ApplicationName,
    SubmitTime,
    State,
    Errors
FROM sys.dm_spark_applications
WHERE SubmitTime >= DATEADD(day, -1, GETDATE())
ORDER BY SubmitTime DESC;
```

### SQL Requests
```sql
SELECT 
    request_id,
    command,
    submit_time,
    start_time,
    end_time,
    status,
    total_elapsed_time
FROM sys.dm_pdw_exec_requests
WHERE submit_time >= DATEADD(hour, -1, GETDATE())
ORDER BY submit_time DESC;
```

---

## Cost Optimization

### DWU Scaling (Dedicated SQL Pool)
```sql
-- Scale up
ALTER DATABASE MyDatabase
MODIFY (SERVICE_OBJECTIVE = 'DW500c');

-- Scale down
ALTER DATABASE MyDatabase
MODIFY (SERVICE_OBJECTIVE = 'DW100c');

-- Pause
ALTER DATABASE MyDatabase PAUSE;

-- Resume
ALTER DATABASE MyDatabase RESUME;
```

### Cost Monitoring
```sql
-- Estimate serverless SQL cost
-- (Data scanned * $5 per TB)
SELECT 
    SUM(data_processed_mb) / 1024 / 1024 AS data_processed_tb,
    SUM(data_processed_mb) / 1024 / 1024 * 5 AS estimated_cost_usd
FROM sys.dm_external_data_processed;
```

---

## Security Best Practices

### SQL Authentication
```sql
-- Create user
CREATE USER [user@domain.com] FROM EXTERNAL PROVIDER;

-- Grant permissions
GRANT SELECT ON SCHEMA::dbo TO [user@domain.com];

-- Row-level security
CREATE SECURITY POLICY dbo.CustomerSecurityPolicy
ADD FILTER PREDICATE dbo.CustomerSecurityPredicate(CustomerID)
ON dbo.Orders
WITH (STATE = ON);
```

### Managed Identity
```python
# Access storage using MI
spark.conf.set(
    f"fs.azure.account.auth.type.{storage_account}.dfs.core.windows.net",
    "OAuth"
)
spark.conf.set(
    f"fs.azure.account.oauth.provider.type.{storage_account}.dfs.core.windows.net",
    "org.apache.hadoop.fs.azurebfs.oauth2.MsiTokenProvider"
)
```

---

## Troubleshooting

### Common Issues

**Issue**: "File not found"
```python
# Solution: Check path and permissions
dbutils.fs.ls("abfss://container@storage.dfs.core.windows.net/")
```

**Issue**: "Out of memory in Spark"
```python
# Solution: Increase executor memory or partition data
spark.conf.set("spark.executor.memory", "8g")
df.repartition(100)
```

**Issue**: "Slow SQL query"
```sql
-- Solution: Check execution plan
EXPLAIN SELECT * FROM LargeTable WHERE Date = '2024-01-01';

-- Create statistics
CREATE STATISTICS stat_date ON LargeTable(Date);

-- Update statistics
UPDATE STATISTICS LargeTable;
```

---

## Useful Links

- **Azure Portal**: https://portal.azure.com
- **Synapse Studio**: https://<workspace>.dev.azuresynapse.net
- **Documentation**: https://docs.microsoft.com/azure/synapse-analytics/
- **Pricing Calculator**: https://azure.microsoft.com/pricing/calculator/
- **Learning Path**: https://docs.microsoft.com/learn/paths/realize-integrated-analytical-solutions-with-azure-synapse-analytics/

---

**Pro Tip**: Bookmark this cheat sheet in Synapse Studio for quick reference!
