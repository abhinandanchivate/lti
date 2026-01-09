
---

# ✅ Combined Hands-On Lab:

## **Synapse + Power BI Reporting, Modelling & Dataflows **

---

## 🎯 Lab Objective 

By the end of this lab, participants will be able to:

* Connect **Power BI to Azure Synapse / Azure data**
* Build **datasets, models, and reports**
* Understand **Import vs DirectQuery**
* Use **Dataflows for data preparation**
* Apply **basic DAX**
* Publish, secure, and optimize Power BI content

---

## 🧱 Architecture Used (Explain Once)

```
Azure Data (Synapse / SQL / Files)
        ↓
Power BI Dataflow (optional shaping)
        ↓
Power BI Dataset (Semantic Model)
        ↓
Power BI Reports & Dashboard
```

---

## 🔹 PART 1: Azure & Synapse Preparation (From Lab 1)

### Step 1: Log in to Azure Portal

1. Open [https://portal.azure.com](https://portal.azure.com)
2. Log in using lab credentials
3. Confirm Azure dashboard loads

---

### Step 2: Create Resource Group (Optional)

1. Search **Resource Groups** → **Create**
2. Enter:

   * **Name:** `Synapse-PBI-RG`
   * **Region:** As per lab instructions
3. Click **Review + Create → Create**

---

### Step 3: Prepare Synapse SQL

1. Open **Azure Synapse Workspace**

2. Go to **Data** → **SQL Pools**

3. Use:

   * **Serverless SQL pool** (preferred)

4. Validate:

   * Tables / views already exist
     (from Parquet / Delta / pipelines / ASA output)

5. Note down:

   * SQL endpoint:
     `<workspace-name>.sql.azuresynapse.net`
   * Database name
   * Authentication type

---

## 🔹 PART 2: Power BI Desktop – Data Connection & Reporting

### Step 4: Open Power BI Desktop

1. Install **Power BI Desktop** (if not installed)
2. Open Power BI Desktop
3. Click **Get Data → Azure → Azure Synapse Analytics**

---

### Step 5: Connect to Synapse SQL

1. Enter:

   * **Server:** `<workspace-name>.sql.azuresynapse.net`
   * **Database:** `<database-name>`
2. Choose connectivity mode:

   * **Import** → better performance
   * **DirectQuery** → real-time data
3. Authenticate using:

   * Azure AD (recommended)
   * SQL Auth (if applicable)

---

### Step 6: Select Tables / Views

1. Browse Synapse objects
2. Select required tables or views
3. Click **Load**
4. Verify data appears in **Fields pane**

---

## 🔹 PART 3: Dataset Publishing & Workspace (Lab 1 + Lab 2 Merge)

### Step 7: Publish Dataset to Power BI Service

1. Click **File → Publish**
2. Select workspace:

   * Personal workspace
   * OR `PBIModellingLab`
3. Confirm publish success

---

### Step 8: Create Power BI Workspace

1. Open [https://app.powerbi.com](https://app.powerbi.com)
2. Go to **Workspaces → + Create**
3. Enter:

   * **Name:** `PBIModellingLab`
4. Assign roles:

   * Admin
   * Member
   * Contributor
   * Viewer
5. Click **Save**

---

## 🔹 PART 4: Dataflows (Lab 2)

### Step 9: Create Dataflow

1. Open **PBIModellingLab workspace**
2. Click **+ Create → Dataflow**
3. Choose **Add new tables**
4. Select source:

   * Azure SQL / Synapse
   * CSV
   * Web (if required)

---

### Step 10: Transform Data (Power Query Online)

1. Rename entities
2. Validate data types
3. Remove unnecessary columns
4. Apply basic transformations
5. Save dataflow

---

### Step 11: Configure Refresh

1. Go to **Dataflow → Settings**
2. Configure:

   * Scheduled refresh
   * Gateway (if private source)
3. Enable **Incremental Refresh** (optional):

   * Historical range: e.g. 1 year
   * Refresh range: e.g. last 7 days

---

## 🔹 PART 5: Dataset Creation & Semantic Modelling

### Step 12: Create Dataset from Dataflow

1. Open Dataflow
2. Click **Create Dataset**
3. Choose:

   * **Import** (recommended for modelling)
4. Confirm dataset creation

---

### Step 13: Model the Data (Power BI Desktop)

1. Open dataset in **Power BI Desktop**
2. Go to **Model view**
3. Create relationships:

   * Fact → Dimension (Star schema)
4. Set:

   * Cardinality
   * Cross-filter direction

---

## 🔹 PART 6: DAX Calculations (Lab 2)

### Step 14: Create Calculated Column

```DAX
Profit = [Revenue] - [Cost]
```

---

### Step 15: Create Measures

```DAX
TotalSales = SUM(Sales[Revenue])

AvgProfit =
CALCULATE(
    AVERAGE(Sales[Profit]),
    Sales[Region] = "North"
)
```

Explain:

* Measures = KPIs
* Calculated columns = row-level logic

---

## 🔹 PART 7: Build Reports & Dashboard

### Step 16: Build Report

1. Add visuals:

   * Line chart (trend)
   * Bar chart (comparison)
   * Card (KPIs)
2. Add slicers:

   * Date
   * Region
3. Save report

---

### Step 17: Create Dashboard

1. Pin visuals to dashboard
2. Arrange tiles
3. Save dashboard in workspace

---

## 🔹 PART 8: Security & Access Control

### Step 18: Row-Level Security (RLS)

1. In Power BI Desktop → **Model → Manage Roles**
2. Add filter:

```DAX
[Region] = USERPRINCIPALNAME()
```

3. Publish report
4. Assign users to roles in Power BI Service

---

### Step 19: Workspace Role Mapping

1. Workspace → **Access**
2. Assign:

   * Admin
   * Member
   * Contributor
   * Viewer
3. Validate permissions

---

## 🔹 PART 9: Refresh & Performance Validation

### Step 20: Validate Refresh

* **Import mode:** Test scheduled refresh
* **DirectQuery:** Validate real-time updates

---

### Step 21: Performance Optimization

1. Remove unused columns
2. Reduce cardinality
3. Use aggregations
4. Enable Incremental Refresh

---

## 🔹 PART 10: End-to-End Validation & Cleanup

### Step 22: End-to-End Validation

1. Verify:

   * Synapse → Power BI data flow
   * Dataflow refresh
   * Dataset refresh
   * Report updates
2. Validate RLS behavior

---

### Step 23: Cleanup (Optional)

1. Delete:

   * Datasets
   * Dataflows
   * Workspace
2. Delete Resource Group (if lab environment)

---


