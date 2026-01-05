# Azure Synapse Analytics - Complete Guide

## Table of Contents
1. [Overview & Architecture](#overview--architecture)
2. [Core Components](#core-components)
3. [Workspace Architecture](#workspace-architecture)
4. [SQL Pools](#sql-pools)
5. [Spark Pools](#spark-pools)
6. [Pipelines & Data Integration](#pipelines--data-integration)
7. [Synapse Studio Navigation](#synapse-studio-navigation)
8. [Databases vs Lake Databases](#databases-vs-lake-databases)
9. [SQL vs Spark Interoperability](#sql-vs-spark-interoperability)
10. [Hands-On Tutorial: Ecommerce Database](#hands-on-tutorial-ecommerce-database)

---

## Overview & Architecture

### What is Azure Synapse Analytics?

Azure Synapse Analytics is a unified analytics platform that combines:
- **Data Integration** (ETL/ELT pipelines)
- **Enterprise Data Warehousing** (SQL)
- **Big Data Analytics** (Spark)
- **Real-time Analytics** (Data Explorer)
- **Visualization** (Power BI integration)

### Key Benefits
- **Unified Experience**: Single workspace for all analytics tasks
- **Serverless & Dedicated Options**: Pay for what you use
- **Security**: Enterprise-grade security with Azure AD, encryption
- **Integration**: Native integration with Azure services
- **Performance**: Massively parallel processing (MPP)

---

## Core Components

### 1. **SQL Pools**

#### Dedicated SQL Pool (formerly SQL DW)
- Provisioned compute and storage
- Best for predictable, high-performance workloads
- Data stored in relational tables with columnar storage
- Pay for provisioned DWU (Data Warehouse Units)

#### Serverless SQL Pool
- On-demand SQL queries
- Query data in Data Lake without loading
- Pay per TB processed
- No infrastructure to manage

### 2. **Apache Spark Pools**
- Distributed computing for big data processing
- Support for Python (PySpark), Scala, .NET, SQL
- Notebooks for interactive development
- Auto-scaling and auto-pause
- Integration with Delta Lake

### 3. **Synapse Pipelines**
- Data orchestration and integration (similar to Azure Data Factory)
- 90+ built-in connectors
- Visual design interface
- Activities: Copy, transformation, control flow
- Triggers: Schedule, tumbling window, event-based

### 4. **Data Explorer Pools (KQL)**
- Real-time analytics on streaming data
- Kusto Query Language (KQL)
- Time-series analysis
- Log analytics and telemetry

---

## Workspace Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     SYNAPSE WORKSPACE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  SQL Pools   │  │ Spark Pools  │  │Data Explorer │          │
│  │              │  │              │  │    Pools     │          │
│  │ • Dedicated  │  │ • Notebooks  │  │              │          │
│  │ • Serverless │  │ • Jobs       │  │ • KQL        │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              SYNAPSE PIPELINES                            │  │
│  │  (Data Integration & Orchestration)                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              SYNAPSE STUDIO                               │  │
│  │  (Unified Development Environment)                        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
            │                    │                    │
            ▼                    ▼                    ▼
    ┌───────────────┐   ┌──────────────┐   ┌──────────────┐
    │  Azure Data   │   │ Azure Blob   │   │ External     │
    │     Lake      │   │   Storage    │   │ Data Sources │
    └───────────────┘   └──────────────┘   └──────────────┘
```

### Key Architecture Components:

1. **Storage Layer**
   - Primary: Azure Data Lake Storage Gen2
   - Hierarchical namespace enabled
   - Containers for different data zones (raw, processed, curated)

2. **Compute Layer**
   - SQL: Dedicated and serverless pools
   - Spark: Configurable cluster sizes
   - Data Explorer: Real-time compute

3. **Integration Layer**
   - Pipelines for data movement
   - Linked services for connectivity
   - Integration runtime (Azure, Self-hosted, Azure-SSIS)

4. **Security Layer**
   - Azure Active Directory
   - Role-Based Access Control (RBAC)
   - Managed Virtual Network
   - Data encryption at rest and in transit

---

## SQL Pools

### Dedicated SQL Pool

**Architecture:**
- Control Node: Query optimization and distribution
- Compute Nodes: Execute queries in parallel
- Storage: Distributed across 60 distributions

**When to Use:**
- Consistent high workload
- Complex queries requiring optimization
- Need for predictable performance
- Data warehousing scenarios

**Performance Levels:**
- DW100c to DW30000c
- Scale up/down based on needs
- Can pause when not in use

**Example: Create Dedicated SQL Pool**
```sql
-- In Azure Portal or via Azure CLI
-- Create a dedicated SQL pool with DW100c

-- Once created, connect and create tables
CREATE TABLE dbo.Products (
    ProductID INT NOT NULL,
    ProductName NVARCHAR(100),
    CategoryID INT,
    UnitPrice DECIMAL(10,2),
    UnitsInStock INT
)
WITH (
    DISTRIBUTION = HASH(ProductID),
    CLUSTERED COLUMNSTORE INDEX
);
```

### Serverless SQL Pool

**Characteristics:**
- Always available (no creation needed)
- Query data directly from Data Lake
- T-SQL language support
- Pay per TB scanned

**When to Use:**
- Ad-hoc queries on data lake
- Data exploration
- Logical data warehouse
- Cost-effective for infrequent queries

**Example: Query Data Lake**
```sql
-- Query Parquet files directly
SELECT 
    OrderID,
    CustomerID,
    OrderDate,
    TotalAmount
FROM OPENROWSET(
    BULK 'https://mystorageaccount.dfs.core.windows.net/data/orders/*.parquet',
    FORMAT = 'PARQUET'
) AS orders
WHERE OrderDate >= '2024-01-01';

-- Create external table for reusability
CREATE EXTERNAL TABLE Orders (
    OrderID INT,
    CustomerID INT,
    OrderDate DATE,
    TotalAmount DECIMAL(10,2)
)
WITH (
    LOCATION = 'orders/',
    DATA_SOURCE = ExternalDataSource,
    FILE_FORMAT = ParquetFormat
);
```

---

## Spark Pools

### Configuration Options

**Node Sizes:**
- Small: 4 vCores, 32 GB RAM
- Medium: 8 vCores, 64 GB RAM
- Large: 16 vCores, 128 GB RAM
- XLarge: 32 vCores, 256 GB RAM

**Features:**
- Auto-scaling: Min to max nodes
- Auto-pause: After idle time (5+ minutes)
- Spark versions: 3.x
- Libraries: Install custom packages

### Spark Pool Architecture

```
┌──────────────────────────────────────┐
│        SPARK CLUSTER                 │
├──────────────────────────────────────┤
│  ┌────────────────────┐              │
│  │   Driver Node      │              │
│  │  • SparkContext    │              │
│  │  • Job Scheduling  │              │
│  └────────────────────┘              │
│           │                           │
│  ┌────────┴────────┐                │
│  │  Worker Nodes   │                │
│  │  • Executors    │                │
│  │  • Task Exec    │                │
│  │  • Cache        │                │
│  └─────────────────┘                │
└──────────────────────────────────────┘
```

### Example: PySpark Code
```python
# Read data from Data Lake
df = spark.read.parquet("abfss://container@storage.dfs.core.windows.net/orders/")

# Data transformations
from pyspark.sql.functions import col, sum, count

order_summary = df.groupBy("CustomerID") \
    .agg(
        count("OrderID").alias("TotalOrders"),
        sum("TotalAmount").alias("TotalSpent")
    ) \
    .orderBy(col("TotalSpent").desc())

# Write results
order_summary.write.mode("overwrite").parquet(
    "abfss://container@storage.dfs.core.windows.net/analytics/customer_summary/"
)
```

---

## Pipelines & Data Integration

### Pipeline Components

1. **Activities**
   - **Data Movement**: Copy activity
   - **Data Transformation**: Data Flow, Spark, SQL
   - **Control Flow**: If, ForEach, Until, Wait
   - **External**: Web, Stored Procedure, Function

2. **Triggers**
   - **Schedule**: Recurrence pattern
   - **Tumbling Window**: Fixed-size, non-overlapping intervals
   - **Event-Based**: Storage events (blob created)
   - **Manual**: On-demand execution

3. **Integration Runtime**
   - **Azure IR**: Managed, serverless
   - **Self-Hosted IR**: On-premises/private networks
   - **Azure-SSIS IR**: Lift-and-shift SSIS packages

### Pipeline Structure

```
Pipeline
├── Activities
│   ├── Copy Data (Source → Sink)
│   ├── Data Flow (Transformations)
│   ├── ForEach (Iterate over items)
│   └── Stored Procedure (Execute SQL)
├── Parameters (Dynamic values)
├── Variables (Runtime storage)
└── Triggers (Execution schedule)
```

### Example: Copy Activity JSON
```json
{
    "name": "CopyOrderData",
    "type": "Copy",
    "inputs": [
        {
            "referenceName": "SourceDataset",
            "type": "DatasetReference"
        }
    ],
    "outputs": [
        {
            "referenceName": "SinkDataset",
            "type": "DatasetReference"
        }
    ],
    "typeProperties": {
        "source": {
            "type": "DelimitedTextSource",
            "storeSettings": {
                "type": "AzureBlobFSReadSettings"
            }
        },
        "sink": {
            "type": "SqlDWSink",
            "writeBehavior": "Insert"
        }
    }
}
```

---

## Synapse Studio Navigation

### Main Hubs

1. **Home Hub** 🏠
   - Quick actions
   - Recent items
   - Learning resources

2. **Data Hub** 📊
   - Workspace databases
   - Lake databases
   - Linked services
   - Browse data lake

3. **Develop Hub** 💻
   - SQL scripts
   - Notebooks (Spark)
   - Data flows
   - Power BI reports

4. **Integrate Hub** 🔄
   - Pipelines
   - Copy data tool
   - Browse templates

5. **Monitor Hub** 📈
   - Pipeline runs
   - Trigger runs
   - Spark applications
   - SQL requests

6. **Manage Hub** ⚙️
   - SQL pools
   - Spark pools
   - Linked services
   - Triggers
   - Access control

### Navigation Flow

```
Synapse Studio
    │
    ├─ Data → Browse/Query data
    │   └─ Create → Lake database/External table
    │
    ├─ Develop → Write code
    │   ├─ SQL Script → Query data
    │   ├─ Notebook → Spark analysis
    │   └─ Data Flow → Visual ETL
    │
    ├─ Integrate → Orchestrate
    │   └─ Pipeline → Schedule jobs
    │
    └─ Monitor → Track execution
        └─ Pipeline runs → Debug issues
```

---

## Databases vs Lake Databases

### Traditional Databases (SQL Pool Databases)

**Characteristics:**
- Data stored in SQL pool
- Structured schema
- Relational model
- ACID transactions
- High performance for queries

**Use Cases:**
- Traditional data warehousing
- Structured data
- Complex joins and aggregations
- Mission-critical queries

### Lake Databases

**Characteristics:**
- Metadata layer over data lake
- Files stored as Parquet/Delta
- Serverless access
- Schema-on-read
- Cost-effective storage

**Use Cases:**
- Big data scenarios
- Semi-structured data
- Data science workloads
- Cost-sensitive projects

### Comparison Table

| Feature | SQL Pool Database | Lake Database |
|---------|-------------------|---------------|
| Storage | SQL Pool | Data Lake (Parquet/Delta) |
| Schema | Schema-on-write | Schema-on-read |
| Query | Dedicated/Serverless SQL | Serverless SQL, Spark |
| Cost | Higher (compute) | Lower (storage) |
| Performance | Optimized | Good for big data |
| Transactions | ACID | Limited |

### Example: Create Lake Database

```sql
-- Create Lake Database
CREATE DATABASE EcommerceLakeDB;

-- Create external data source
CREATE EXTERNAL DATA SOURCE LakeDataSource
WITH (
    LOCATION = 'abfss://data@mystorageaccount.dfs.core.windows.net'
);

-- Create external file format
CREATE EXTERNAL FILE FORMAT ParquetFileFormat
WITH (
    FORMAT_TYPE = PARQUET,
    DATA_COMPRESSION = 'org.apache.hadoop.io.compress.SnappyCodec'
);

-- Create external table (Lake Table)
CREATE EXTERNAL TABLE dbo.Orders (
    OrderID INT,
    CustomerID INT,
    OrderDate DATE,
    TotalAmount DECIMAL(10,2)
)
WITH (
    LOCATION = '/orders/',
    DATA_SOURCE = LakeDataSource,
    FILE_FORMAT = ParquetFileFormat
);
```

---

## SQL vs Spark Interoperability

### Data Exchange Patterns

1. **SQL → Spark**
   - Read SQL tables in Spark
   - Query data with SparkSQL

2. **Spark → SQL**
   - Write Spark DataFrames to SQL tables
   - Create external tables over Spark output

3. **Shared Data Lake**
   - Both access same files
   - SQL reads what Spark writes

### Example: SQL to Spark

```python
# PySpark: Read from SQL Pool
df = spark.read \
    .format("com.microsoft.sqlserver.jdbc.spark") \
    .option("url", "jdbc:sqlserver://workspace.sql.azuresynapse.net:1433") \
    .option("dbtable", "dbo.Orders") \
    .option("user", "sqladmin") \
    .option("password", "P@ssw0rd") \
    .load()

# Process in Spark
result = df.filter(col("OrderDate") >= "2024-01-01")

# Write back to SQL pool
result.write \
    .format("com.microsoft.sqlserver.jdbc.spark") \
    .option("url", "jdbc:sqlserver://workspace.sql.azuresynapse.net:1433") \
    .option("dbtable", "dbo.OrdersFiltered") \
    .option("user", "sqladmin") \
    .option("password", "P@ssw0rd") \
    .mode("overwrite") \
    .save()
```

### Example: Spark to SQL

```python
# Write Spark DataFrame to Data Lake
df.write.mode("overwrite").parquet(
    "abfss://container@storage.dfs.core.windows.net/processed/orders/"
)

# Then in SQL (Serverless or Dedicated):
# Create external table pointing to Spark output
```

```sql
CREATE EXTERNAL TABLE OrdersProcessed (
    OrderID INT,
    CustomerID INT,
    OrderDate DATE,
    TotalAmount DECIMAL(10,2)
)
WITH (
    LOCATION = '/processed/orders/',
    DATA_SOURCE = ExternalDataSource,
    FILE_FORMAT = ParquetFormat
);

-- Query Spark output via SQL
SELECT * FROM OrdersProcessed;
```

---

## Hands-On Tutorial: Ecommerce Database

### Scenario Overview

We'll build a complete analytics solution for an ecommerce company:
- **Data**: Orders, Customers, Products, Order Details
- **Storage**: Azure Data Lake Storage Gen2
- **Processing**: SQL and Spark
- **Orchestration**: Synapse Pipelines

### Prerequisites

- Azure subscription
- Resource group created
- Azure Data Lake Storage Gen2 account

---

## Step 1: Create Synapse Workspace

### Via Azure Portal

1. **Navigate to Azure Portal**
   - Search for "Azure Synapse Analytics"
   - Click "+ Create"

2. **Basics Tab**
   - **Subscription**: Select your subscription
   - **Resource Group**: Create new or select existing
   - **Workspace name**: `ecommerce-synapse-workspace`
   - **Region**: Choose your region
   - **Data Lake Storage Gen2**:
     - Account name: `ecommerceadls` (create new)
     - File system: `data` (create new)

3. **Security Tab**
   - **SQL admin login**: `sqladmin`
   - **Password**: Create strong password
   - Enable: "Allow connections from all IP addresses" (for testing)

4. **Review + Create**
   - Validate and create (takes 5-10 minutes)

### Via Azure CLI

```bash
# Variables
RESOURCE_GROUP="rg-ecommerce-synapse"
LOCATION="eastus"
STORAGE_ACCOUNT="ecommerceadls"
FILE_SYSTEM="data"
WORKSPACE_NAME="ecommerce-synapse-workspace"

# Create resource group
az group create --name $RESOURCE_GROUP --location $LOCATION

# Create storage account
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2 \
  --hierarchical-namespace true

# Create file system
az storage fs create \
  --name $FILE_SYSTEM \
  --account-name $STORAGE_ACCOUNT

# Create Synapse workspace
az synapse workspace create \
  --name $WORKSPACE_NAME \
  --resource-group $RESOURCE_GROUP \
  --storage-account $STORAGE_ACCOUNT \
  --file-system $FILE_SYSTEM \
  --sql-admin-login-user sqladmin \
  --sql-admin-login-password 'YourP@ssw0rd123' \
  --location $LOCATION

# Allow access from your IP
az synapse workspace firewall-rule create \
  --name AllowAll \
  --workspace-name $WORKSPACE_NAME \
  --resource-group $RESOURCE_GROUP \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 255.255.255.255
```

---

## Step 2: Prepare Sample Ecommerce Data

### Create Sample CSV Files

**customers.csv**
```csv
CustomerID,FirstName,LastName,Email,Country,City,RegistrationDate
1,John,Smith,john.smith@email.com,USA,New York,2023-01-15
2,Emma,Johnson,emma.j@email.com,UK,London,2023-02-20
3,Michael,Brown,m.brown@email.com,USA,Los Angeles,2023-03-10
4,Sophia,Davis,sophia.d@email.com,Canada,Toronto,2023-04-05
5,James,Wilson,james.w@email.com,USA,Chicago,2023-05-12
```

**products.csv**
```csv
ProductID,ProductName,Category,UnitPrice,UnitsInStock
101,Laptop Pro 15,Electronics,1299.99,50
102,Wireless Mouse,Electronics,29.99,200
103,USB-C Cable,Accessories,15.99,500
104,Monitor 27inch,Electronics,349.99,75
105,Keyboard Mechanical,Electronics,89.99,120
106,Laptop Bag,Accessories,49.99,150
107,Webcam HD,Electronics,79.99,100
108,Headphones,Electronics,129.99,180
```

**orders.csv**
```csv
OrderID,CustomerID,OrderDate,ShipDate,Status,TotalAmount
1001,1,2024-01-10,2024-01-12,Delivered,1379.98
1002,2,2024-01-11,2024-01-14,Delivered,29.99
1003,3,2024-01-12,2024-01-15,Delivered,399.98
1004,1,2024-01-15,2024-01-18,Delivered,145.98
1005,4,2024-01-18,2024-01-21,Delivered,1429.97
1006,5,2024-01-20,2024-01-23,Delivered,349.99
1007,2,2024-01-22,2024-01-25,Delivered,219.97
1008,3,2024-01-25,NULL,Shipped,89.99
```

**order_details.csv**
```csv
OrderDetailID,OrderID,ProductID,Quantity,UnitPrice,Discount
1,1001,101,1,1299.99,0.0
2,1001,103,5,15.99,0.1
3,1002,102,1,29.99,0.0
4,1003,104,1,349.99,0.0
5,1003,103,3,15.99,0.05
6,1004,107,1,79.99,0.0
7,1004,108,1,129.99,0.0
8,1005,101,1,1299.99,0.0
9,1005,108,1,129.99,0.0
10,1006,104,1,349.99,0.0
11,1007,105,1,89.99,0.0
12,1007,108,1,129.99,0.0
13,1008,105,1,89.99,0.0
```

### Upload to Data Lake

**Via Azure Portal:**

1. Navigate to your storage account `ecommerceadls`
2. Go to "Containers" → Select `data` container
3. Create folder structure:
   ```
   data/
   ├── raw/
   │   └── ecommerce/
   │       ├── customers/
   │       ├── products/
   │       ├── orders/
   │       └── order_details/
   ```
4. Upload CSV files to respective folders

**Via Azure CLI:**

```bash
STORAGE_ACCOUNT="ecommerceadls"

# Upload files
az storage blob upload --account-name $STORAGE_ACCOUNT \
  --container-name data --file customers.csv \
  --name raw/ecommerce/customers/customers.csv

az storage blob upload --account-name $STORAGE_ACCOUNT \
  --container-name data --file products.csv \
  --name raw/ecommerce/products/products.csv

az storage blob upload --account-name $STORAGE_ACCOUNT \
  --container-name data --file orders.csv \
  --name raw/ecommerce/orders/orders.csv

az storage blob upload --account-name $STORAGE_ACCOUNT \
  --container-name data --file order_details.csv \
  --name raw/ecommerce/order_details/order_details.csv
```

---

## Step 3: Create Spark Pool

### Via Synapse Studio

1. **Open Synapse Studio**
   - From workspace overview, click "Open Synapse Studio"

2. **Navigate to Manage Hub** ⚙️
   - Click "Manage" from left menu
   - Select "Apache Spark pools"

3. **Create New Spark Pool**
   - Click "+ New"
   - **Name**: `ecommercesparkpool`
   - **Node size**: Small (4 vCores, 32 GB)
   - **Autoscale**: Enabled
   - **Min nodes**: 3
   - **Max nodes**: 10
   - **Auto-pause**: Enabled (15 minutes)
   - Click "Review + Create"

### Via Azure CLI

```bash
az synapse spark pool create \
  --name ecommercesparkpool \
  --workspace-name $WORKSPACE_NAME \
  --resource-group $RESOURCE_GROUP \
  --node-count 3 \
  --node-size Small \
  --spark-version 3.3 \
  --enable-auto-scale true \
  --max-node-count 10 \
  --enable-auto-pause true \
  --delay 15
```

---

## Step 4: Query Data with Serverless SQL

### Create SQL Script

1. **Navigate to Develop Hub** 💻
2. Click "+ " → "SQL script"
3. Name: "Query_Raw_Ecommerce_Data"

### Query Customers Data

```sql
-- Query customers CSV file
SELECT 
    CustomerID,
    FirstName,
    LastName,
    Email,
    Country,
    City,
    RegistrationDate
FROM OPENROWSET(
    BULK 'https://ecommerceadls.dfs.core.windows.net/data/raw/ecommerce/customers/*.csv',
    FORMAT = 'CSV',
    PARSER_VERSION = '2.0',
    HEADER_ROW = TRUE
) AS customers;
```

### Query Orders with JOIN

```sql
-- Join orders with customers
SELECT 
    o.OrderID,
    c.FirstName + ' ' + c.LastName AS CustomerName,
    o.OrderDate,
    o.Status,
    o.TotalAmount
FROM OPENROWSET(
    BULK 'https://ecommerceadls.dfs.core.windows.net/data/raw/ecommerce/orders/*.csv',
    FORMAT = 'CSV',
    PARSER_VERSION = '2.0',
    HEADER_ROW = TRUE
) AS o
INNER JOIN OPENROWSET(
    BULK 'https://ecommerceadls.dfs.core.windows.net/data/raw/ecommerce/customers/*.csv',
    FORMAT = 'CSV',
    PARSER_VERSION = '2.0',
    HEADER_ROW = TRUE
) AS c
ON o.CustomerID = c.CustomerID
ORDER BY o.OrderDate DESC;
```

### Create External Tables for Reusability

```sql
-- Create Database
CREATE DATABASE EcommerceDB;
GO

USE EcommerceDB;
GO

-- Create external data source
CREATE EXTERNAL DATA SOURCE EcommerceDataLake
WITH (
    LOCATION = 'https://ecommerceadls.dfs.core.windows.net/data/raw/ecommerce'
);
GO

-- Create external file format
CREATE EXTERNAL FILE FORMAT CsvFormat
WITH (
    FORMAT_TYPE = DELIMITEDTEXT,
    FORMAT_OPTIONS (
        FIELD_TERMINATOR = ',',
        STRING_DELIMITER = '"',
        FIRST_ROW = 2,
        USE_TYPE_DEFAULT = FALSE,
        ENCODING = 'UTF8',
        PARSER_VERSION = '2.0'
    )
);
GO

-- Create external table: Customers
CREATE EXTERNAL TABLE Customers (
    CustomerID INT,
    FirstName NVARCHAR(50),
    LastName NVARCHAR(50),
    Email NVARCHAR(100),
    Country NVARCHAR(50),
    City NVARCHAR(50),
    RegistrationDate DATE
)
WITH (
    LOCATION = 'customers/',
    DATA_SOURCE = EcommerceDataLake,
    FILE_FORMAT = CsvFormat
);
GO

-- Create external table: Products
CREATE EXTERNAL TABLE Products (
    ProductID INT,
    ProductName NVARCHAR(100),
    Category NVARCHAR(50),
    UnitPrice DECIMAL(10,2),
    UnitsInStock INT
)
WITH (
    LOCATION = 'products/',
    DATA_SOURCE = EcommerceDataLake,
    FILE_FORMAT = CsvFormat
);
GO

-- Create external table: Orders
CREATE EXTERNAL TABLE Orders (
    OrderID INT,
    CustomerID INT,
    OrderDate DATE,
    ShipDate DATE,
    Status NVARCHAR(20),
    TotalAmount DECIMAL(10,2)
)
WITH (
    LOCATION = 'orders/',
    DATA_SOURCE = EcommerceDataLake,
    FILE_FORMAT = CsvFormat
);
GO

-- Create external table: OrderDetails
CREATE EXTERNAL TABLE OrderDetails (
    OrderDetailID INT,
    OrderID INT,
    ProductID INT,
    Quantity INT,
    UnitPrice DECIMAL(10,2),
    Discount DECIMAL(3,2)
)
WITH (
    LOCATION = 'order_details/',
    DATA_SOURCE = EcommerceDataLake,
    FILE_FORMAT = CsvFormat
);
GO

-- Test queries
SELECT * FROM Customers;
SELECT * FROM Products;
SELECT * FROM Orders;
SELECT * FROM OrderDetails;
```

### Business Queries

```sql
-- 1. Total sales by country
SELECT 
    c.Country,
    COUNT(DISTINCT o.OrderID) AS TotalOrders,
    SUM(o.TotalAmount) AS TotalRevenue,
    AVG(o.TotalAmount) AS AvgOrderValue
FROM Orders o
INNER JOIN Customers c ON o.CustomerID = c.CustomerID
GROUP BY c.Country
ORDER BY TotalRevenue DESC;

-- 2. Top selling products
SELECT 
    p.ProductName,
    p.Category,
    SUM(od.Quantity) AS TotalQuantitySold,
    SUM(od.Quantity * od.UnitPrice * (1 - od.Discount)) AS TotalRevenue
FROM OrderDetails od
INNER JOIN Products p ON od.ProductID = p.ProductID
GROUP BY p.ProductName, p.Category
ORDER BY TotalRevenue DESC;

-- 3. Customer lifetime value
SELECT 
    c.CustomerID,
    c.FirstName + ' ' + c.LastName AS CustomerName,
    COUNT(o.OrderID) AS TotalOrders,
    SUM(o.TotalAmount) AS LifetimeValue,
    MIN(o.OrderDate) AS FirstOrderDate,
    MAX(o.OrderDate) AS LastOrderDate
FROM Customers c
LEFT JOIN Orders o ON c.CustomerID = o.CustomerID
GROUP BY c.CustomerID, c.FirstName, c.LastName
ORDER BY LifetimeValue DESC;

-- 4. Monthly sales trend
SELECT 
    YEAR(OrderDate) AS OrderYear,
    MONTH(OrderDate) AS OrderMonth,
    COUNT(OrderID) AS TotalOrders,
    SUM(TotalAmount) AS MonthlyRevenue
FROM Orders
GROUP BY YEAR(OrderDate), MONTH(OrderDate)
ORDER BY OrderYear, OrderMonth;
```

---

## Step 5: Process Data with Spark

### Create Spark Notebook

1. **Navigate to Develop Hub** 💻
2. Click "+ " → "Notebook"
3. **Name**: `Ecommerce_Spark_Analysis`
4. **Attach to**: Select `ecommercesparkpool`

### Notebook Cell 1: Read Data

```python
# Read CSV files from Data Lake
from pyspark.sql.functions import *
from pyspark.sql.types import *

# Define storage path
storage_account = "ecommerceadls"
container = "data"
base_path = f"abfss://{container}@{storage_account}.dfs.core.windows.net/raw/ecommerce"

# Read customers
customers_df = spark.read \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .csv(f"{base_path}/customers/customers.csv")

# Read products
products_df = spark.read \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .csv(f"{base_path}/products/products.csv")

# Read orders
orders_df = spark.read \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .csv(f"{base_path}/orders/orders.csv")

# Read order details
order_details_df = spark.read \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .csv(f"{base_path}/order_details/order_details.csv")

# Display sample data
print("Customers:")
customers_df.show(5)

print("Products:")
products_df.show(5)

print("Orders:")
orders_df.show(5)

print("Order Details:")
order_details_df.show(5)
```

### Notebook Cell 2: Data Quality Checks

```python
# Check for nulls
print("=== NULL CHECK ===")
print(f"Customers nulls: {customers_df.filter(col('CustomerID').isNull()).count()}")
print(f"Products nulls: {products_df.filter(col('ProductID').isNull()).count()}")
print(f"Orders nulls: {orders_df.filter(col('OrderID').isNull()).count()}")
print(f"OrderDetails nulls: {order_details_df.filter(col('OrderDetailID').isNull()).count()}")

# Check record counts
print("\n=== RECORD COUNTS ===")
print(f"Customers: {customers_df.count()}")
print(f"Products: {products_df.count()}")
print(f"Orders: {orders_df.count()}")
print(f"OrderDetails: {order_details_df.count()}")

# Check data types
print("\n=== CUSTOMERS SCHEMA ===")
customers_df.printSchema()
```

### Notebook Cell 3: Transform Data

```python
# Add derived columns
orders_enriched = orders_df \
    .withColumn("OrderYear", year(col("OrderDate"))) \
    .withColumn("OrderMonth", month(col("OrderDate"))) \
    .withColumn("OrderQuarter", quarter(col("OrderDate"))) \
    .withColumn("DaysToShip", datediff(col("ShipDate"), col("OrderDate")))

# Calculate order metrics
order_metrics = order_details_df \
    .withColumn("LineTotal", col("Quantity") * col("UnitPrice") * (1 - col("Discount"))) \
    .groupBy("OrderID") \
    .agg(
        count("OrderDetailID").alias("ItemCount"),
        sum("LineTotal").alias("OrderTotal"),
        avg("LineTotal").alias("AvgLineTotal")
    )

# Join with orders
orders_with_metrics = orders_enriched.join(order_metrics, "OrderID", "left")

orders_with_metrics.show(10)
```

### Notebook Cell 4: Customer Analytics

```python
# Customer order summary
customer_summary = orders_df \
    .join(customers_df, "CustomerID") \
    .groupBy("CustomerID", "FirstName", "LastName", "Email", "Country") \
    .agg(
        count("OrderID").alias("TotalOrders"),
        sum("TotalAmount").alias("TotalSpent"),
        avg("TotalAmount").alias("AvgOrderValue"),
        min("OrderDate").alias("FirstOrderDate"),
        max("OrderDate").alias("LastOrderDate")
    ) \
    .withColumn("CustomerTenure", datediff(col("LastOrderDate"), col("FirstOrderDate"))) \
    .orderBy(col("TotalSpent").desc())

# Customer segmentation
customer_segments = customer_summary \
    .withColumn("Segment", 
        when(col("TotalSpent") >= 1000, "High Value")
        .when(col("TotalSpent") >= 500, "Medium Value")
        .otherwise("Low Value")
    )

print("Customer Segments:")
customer_segments.show(10)

# Segment distribution
segment_distribution = customer_segments \
    .groupBy("Segment") \
    .agg(
        count("CustomerID").alias("CustomerCount"),
        sum("TotalSpent").alias("SegmentRevenue"),
        avg("TotalSpent").alias("AvgCustomerValue")
    )

segment_distribution.show()
```

### Notebook Cell 5: Product Analytics

```python
# Product performance
product_performance = order_details_df \
    .join(products_df, "ProductID") \
    .join(orders_df, "OrderID") \
    .groupBy("ProductID", "ProductName", "Category") \
    .agg(
        count("OrderDetailID").alias("TimesSold"),
        sum("Quantity").alias("TotalQuantitySold"),
        sum(col("Quantity") * col("UnitPrice") * (1 - col("Discount"))).alias("TotalRevenue"),
        avg(col("UnitPrice")).alias("AvgSellingPrice")
    ) \
    .orderBy(col("TotalRevenue").desc())

print("Top Products by Revenue:")
product_performance.show(10)

# Category performance
category_performance = product_performance \
    .groupBy("Category") \
    .agg(
        count("ProductID").alias("ProductCount"),
        sum("TotalQuantitySold").alias("TotalUnitsSold"),
        sum("TotalRevenue").alias("CategoryRevenue")
    ) \
    .orderBy(col("CategoryRevenue").desc())

print("\nCategory Performance:")
category_performance.show()
```

### Notebook Cell 6: Sales Trends

```python
# Monthly sales trend
monthly_sales = orders_df \
    .withColumn("YearMonth", date_format(col("OrderDate"), "yyyy-MM")) \
    .groupBy("YearMonth") \
    .agg(
        count("OrderID").alias("OrderCount"),
        sum("TotalAmount").alias("Revenue"),
        avg("TotalAmount").alias("AvgOrderValue")
    ) \
    .orderBy("YearMonth")

print("Monthly Sales Trend:")
monthly_sales.show()

# Country performance
country_performance = orders_df \
    .join(customers_df, "CustomerID") \
    .groupBy("Country") \
    .agg(
        countDistinct("CustomerID").alias("UniqueCustomers"),
        count("OrderID").alias("TotalOrders"),
        sum("TotalAmount").alias("TotalRevenue")
    ) \
    .orderBy(col("TotalRevenue").desc())

print("\nCountry Performance:")
country_performance.show()
```

### Notebook Cell 7: Write Results to Data Lake

```python
# Define output path
output_path = f"abfss://{container}@{storage_account}.dfs.core.windows.net/processed/ecommerce"

# Write customer segments
customer_segments.write.mode("overwrite").parquet(f"{output_path}/customer_segments")

# Write product performance
product_performance.write.mode("overwrite").parquet(f"{output_path}/product_performance")

# Write monthly sales
monthly_sales.write.mode("overwrite").parquet(f"{output_path}/monthly_sales")

# Write enriched orders
orders_with_metrics.write.mode("overwrite").parquet(f"{output_path}/orders_enriched")

print("✓ Data successfully written to processed layer")

# Create temporary views for SQL querying
customer_segments.createOrReplaceTempView("customer_segments_view")
product_performance.createOrReplaceTempView("product_performance_view")
monthly_sales.createOrReplaceTempView("monthly_sales_view")

print("✓ Temporary views created")
```

### Notebook Cell 8: Query with Spark SQL

```python
# Run Spark SQL queries
result = spark.sql("""
    SELECT 
        Segment,
        COUNT(*) as CustomerCount,
        ROUND(SUM(TotalSpent), 2) as TotalRevenue,
        ROUND(AVG(TotalSpent), 2) as AvgCustomerValue
    FROM customer_segments_view
    GROUP BY Segment
    ORDER BY TotalRevenue DESC
""")

result.show()

# Top 5 products
top_products = spark.sql("""
    SELECT 
        ProductName,
        Category,
        TotalRevenue,
        TotalQuantitySold
    FROM product_performance_view
    ORDER BY TotalRevenue DESC
    LIMIT 5
""")

print("\nTop 5 Products:")
top_products.show()
```

---

## Step 6: Create Data Integration Pipeline

### Pipeline: Copy Data from CSV to Parquet

1. **Navigate to Integrate Hub** 🔄
2. Click "+ " → "Pipeline"
3. **Name**: `Pipeline_CSV_to_Parquet`

### Add Copy Data Activities

**Activity 1: Copy Customers**

1. **Drag "Copy data" activity** to canvas
2. **Name**: `Copy_Customers`
3. **Source tab**:
   - Create new dataset
   - Type: Azure Data Lake Storage Gen2
   - Format: DelimitedText (CSV)
   - Path: `data/raw/ecommerce/customers`
   - First row as header: Yes

4. **Sink tab**:
   - Create new dataset
   - Type: Azure Data Lake Storage Gen2
   - Format: Parquet
   - Path: `data/processed/ecommerce/customers`

**Repeat for other tables** (Products, Orders, OrderDetails)

### Add Notebook Activity

1. **Drag "Notebook" activity** to canvas
2. **Name**: `Run_Spark_Analysis`
3. **Settings tab**:
   - **Notebook**: Select `Ecommerce_Spark_Analysis`
   - **Spark pool**: `ecommercesparkpool`
4. **Connect** Copy activities → Notebook (Success path)

### Add Stored Procedure Activity

1. **Drag "Stored procedure" activity** to canvas
2. **Name**: `Update_Metadata_Table`
3. Connect Notebook → Stored Procedure

### Complete Pipeline JSON

```json
{
    "name": "Pipeline_CSV_to_Parquet",
    "properties": {
        "activities": [
            {
                "name": "Copy_Customers",
                "type": "Copy",
                "dependsOn": [],
                "policy": {
                    "timeout": "0.12:00:00",
                    "retry": 0
                },
                "typeProperties": {
                    "source": {
                        "type": "DelimitedTextSource",
                        "storeSettings": {
                            "type": "AzureBlobFSReadSettings",
                            "recursive": true
                        }
                    },
                    "sink": {
                        "type": "ParquetSink",
                        "storeSettings": {
                            "type": "AzureBlobFSWriteSettings"
                        }
                    },
                    "enableStaging": false
                }
            },
            {
                "name": "Copy_Products",
                "type": "Copy",
                "dependsOn": [],
                "typeProperties": {
                    "source": {
                        "type": "DelimitedTextSource"
                    },
                    "sink": {
                        "type": "ParquetSink"
                    }
                }
            },
            {
                "name": "Copy_Orders",
                "type": "Copy",
                "dependsOn": [],
                "typeProperties": {
                    "source": {
                        "type": "DelimitedTextSource"
                    },
                    "sink": {
                        "type": "ParquetSink"
                    }
                }
            },
            {
                "name": "Copy_OrderDetails",
                "type": "Copy",
                "dependsOn": [],
                "typeProperties": {
                    "source": {
                        "type": "DelimitedTextSource"
                    },
                    "sink": {
                        "type": "ParquetSink"
                    }
                }
            },
            {
                "name": "Run_Spark_Analysis",
                "type": "SynapseNotebook",
                "dependsOn": [
                    {
                        "activity": "Copy_Customers",
                        "dependencyConditions": ["Succeeded"]
                    },
                    {
                        "activity": "Copy_Products",
                        "dependencyConditions": ["Succeeded"]
                    },
                    {
                        "activity": "Copy_Orders",
                        "dependencyConditions": ["Succeeded"]
                    },
                    {
                        "activity": "Copy_OrderDetails",
                        "dependencyConditions": ["Succeeded"]
                    }
                ],
                "policy": {
                    "timeout": "0.12:00:00",
                    "retry": 0
                },
                "typeProperties": {
                    "notebook": {
                        "referenceName": "Ecommerce_Spark_Analysis",
                        "type": "NotebookReference"
                    },
                    "sparkPool": {
                        "referenceName": "ecommercesparkpool",
                        "type": "BigDataPoolReference"
                    }
                }
            }
        ]
    }
}
```

### Add Trigger

1. **Click "Add trigger"** → "New/Edit"
2. **Trigger type**: Schedule
3. **Name**: `Daily_Ecommerce_Processing`
4. **Recurrence**: 
   - Every: 1 Day
   - Start time: 02:00 AM
   - Time zone: Your timezone
5. **Click OK**

### Test Pipeline

1. Click "**Debug**" to test pipeline
2. Monitor execution in **Monitor Hub**
3. View detailed logs for each activity

---

## Step 7: Query Processed Data

### Query Parquet Files (Serverless SQL)

```sql
-- Query customer segments
SELECT 
    Segment,
    COUNT(*) as CustomerCount,
    SUM(TotalSpent) as TotalRevenue
FROM OPENROWSET(
    BULK 'https://ecommerceadls.dfs.core.windows.net/data/processed/ecommerce/customer_segments/*.parquet',
    FORMAT = 'PARQUET'
) AS segments
GROUP BY Segment;

-- Query product performance
SELECT TOP 10
    ProductName,
    Category,
    TotalRevenue,
    TotalQuantitySold
FROM OPENROWSET(
    BULK 'https://ecommerceadls.dfs.core.windows.net/data/processed/ecommerce/product_performance/*.parquet',
    FORMAT = 'PARQUET'
) AS products
ORDER BY TotalRevenue DESC;

-- Query monthly trends
SELECT 
    YearMonth,
    OrderCount,
    Revenue,
    AvgOrderValue
FROM OPENROWSET(
    BULK 'https://ecommerceadls.dfs.core.windows.net/data/processed/ecommerce/monthly_sales/*.parquet',
    FORMAT = 'PARQUET'
) AS monthly
ORDER BY YearMonth;
```

---

## Advanced Topics

### 1. Dedicated SQL Pool for High Performance

```sql
-- Create dedicated SQL pool (via portal or CLI)

-- Once created, create optimized tables
CREATE TABLE dbo.FactOrders (
    OrderID INT NOT NULL,
    CustomerID INT NOT NULL,
    OrderDate DATE NOT NULL,
    TotalAmount DECIMAL(10,2),
    Status NVARCHAR(20)
)
WITH (
    DISTRIBUTION = HASH(OrderID),
    CLUSTERED COLUMNSTORE INDEX
);

-- Load data from Data Lake
COPY INTO dbo.FactOrders
FROM 'https://ecommerceadls.dfs.core.windows.net/data/processed/ecommerce/orders_enriched/*.parquet'
WITH (
    FILE_TYPE = 'PARQUET',
    CREDENTIAL = (IDENTITY = 'Managed Identity')
);

-- Create partitioned table
CREATE TABLE dbo.FactOrdersPartitioned (
    OrderID INT NOT NULL,
    CustomerID INT NOT NULL,
    OrderDate DATE NOT NULL,
    TotalAmount DECIMAL(10,2)
)
WITH (
    DISTRIBUTION = HASH(OrderID),
    PARTITION (OrderDate RANGE RIGHT FOR VALUES 
        ('2024-01-01', '2024-02-01', '2024-03-01')
    ),
    CLUSTERED COLUMNSTORE INDEX
);
```

### 2. Delta Lake for ACID Transactions

```python
# Write to Delta format
orders_with_metrics.write \
    .format("delta") \
    .mode("overwrite") \
    .save(f"{output_path}/orders_delta")

# Read Delta table
delta_df = spark.read.format("delta").load(f"{output_path}/orders_delta")

# Update Delta table (UPSERT)
from delta.tables import DeltaTable

delta_table = DeltaTable.forPath(spark, f"{output_path}/orders_delta")

# Merge new data
delta_table.alias("target").merge(
    new_orders_df.alias("source"),
    "target.OrderID = source.OrderID"
).whenMatchedUpdate(set = {
    "Status": "source.Status",
    "TotalAmount": "source.TotalAmount"
}).whenNotMatchedInsertAll().execute()

# Time travel
historical_df = spark.read \
    .format("delta") \
    .option("versionAsOf", 0) \
    .load(f"{output_path}/orders_delta")
```

### 3. Pipeline with Parameters

```json
{
    "name": "Pipeline_Parameterized",
    "properties": {
        "parameters": {
            "SourcePath": {
                "type": "string",
                "defaultValue": "raw/ecommerce"
            },
            "TargetPath": {
                "type": "string",
                "defaultValue": "processed/ecommerce"
            },
            "ProcessingDate": {
                "type": "string",
                "defaultValue": "@utcnow()"
            }
        },
        "activities": [
            {
                "name": "CopyWithParams",
                "type": "Copy",
                "typeProperties": {
                    "source": {
                        "type": "DelimitedTextSource"
                    },
                    "sink": {
                        "type": "ParquetSink"
                    }
                },
                "inputs": [{
                    "parameters": {
                        "path": "@pipeline().parameters.SourcePath"
                    }
                }],
                "outputs": [{
                    "parameters": {
                        "path": "@concat(pipeline().parameters.TargetPath, '/', formatDateTime(pipeline().parameters.ProcessingDate, 'yyyy-MM-dd'))"
                    }
                }]
            }
        ]
    }
}
```

### 4. Monitoring and Alerts

```sql
-- Query to monitor pipeline runs
SELECT 
    RunId,
    PipelineName,
    RunStart,
    RunEnd,
    DurationInMs,
    Status
FROM sys.dm_pipeline_runs
WHERE PipelineName = 'Pipeline_CSV_to_Parquet'
ORDER BY RunStart DESC;

-- Query to monitor Spark applications
SELECT 
    ApplicationId,
    ApplicationName,
    SubmitTime,
    State,
    Errors
FROM sys.dm_spark_applications
WHERE State = 'Failed'
ORDER BY SubmitTime DESC;
```

---

## Best Practices

### 1. Data Organization
```
data/
├── raw/              # Original ingested data
├── processed/        # Cleaned and transformed
├── curated/          # Business-ready aggregations
└── archive/          # Historical data
```

### 2. Naming Conventions
- **SQL Pools**: `dedicatedsql01`, `serverlesssql`
- **Spark Pools**: `sparkpool01`, `sparkpoolsmall`
- **Pipelines**: `Pipeline_Source_to_Target`
- **Datasets**: `Dataset_Source_Format`

### 3. Performance Optimization
- Use **distribution strategies** (Hash, Round-robin, Replicate)
- Implement **partitioning** for large tables
- Use **columnstore indexes** for analytics
- Enable **result set caching** for repeated queries
- Use **Spark broadcast joins** for small tables

### 4. Security
- Enable **Azure AD authentication**
- Use **Managed Identities** for resource access
- Implement **row-level security** in SQL
- Enable **column-level encryption**
- Use **private endpoints** for network isolation

### 5. Cost Management
- **Pause** dedicated SQL pools when not in use
- Use **serverless SQL** for ad-hoc queries
- Enable **auto-pause** for Spark pools
- Set **budget alerts** in Azure Cost Management
- Use **appropriate compute sizes**

---

## Summary

You've learned:

✅ **Azure Synapse Architecture** - Unified analytics platform components  
✅ **SQL Pools** - Dedicated vs Serverless for different workloads  
✅ **Spark Pools** - Big data processing with notebooks  
✅ **Pipelines** - Orchestrating data integration workflows  
✅ **Synapse Studio** - Unified development environment  
✅ **Lake Databases** - Metadata layer over data lake  
✅ **SQL-Spark Interoperability** - Seamless data exchange  
✅ **Hands-On** - Built complete ecommerce analytics solution  

### Next Steps

1. **Advanced Analytics**: Implement machine learning with Spark MLlib
2. **Real-time**: Add Data Explorer for streaming analytics
3. **Visualization**: Connect Power BI for dashboards
4. **CI/CD**: Implement DevOps for Synapse artifacts
5. **Governance**: Add Azure Purview for data catalog

---

## Additional Resources

- [Azure Synapse Documentation](https://docs.microsoft.com/azure/synapse-analytics/)
- [Synapse SQL Best Practices](https://docs.microsoft.com/azure/synapse-analytics/sql/best-practices-dedicated-sql-pool)
- [Spark Best Practices](https://docs.microsoft.com/azure/synapse-analytics/spark/apache-spark-best-practices)
- [Pipeline Activities Reference](https://docs.microsoft.com/azure/synapse-analytics/synapse-pipelines/concept-pipeline-activities)
- [Azure Synapse Learning Path](https://docs.microsoft.com/learn/paths/realize-integrated-analytical-solutions-with-azure-synapse-analytics/)

---

**End of Guide**
