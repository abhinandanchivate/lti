# Azure SQL Database Partitioning with Azure Data Studio - Complete Guide

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Azure Naming Conventions](#azure-naming-conventions)
3. [Creating Azure SQL Database Server](#part-1-creating-azure-sql-database-server)
4. [Creating Windows Server VM and Installing Azure Data Studio](#part-2-creating-windows-server-vm-and-installing-azure-data-studio)
5. [Connecting to Azure SQL Database](#part-3-connecting-to-azure-sql-database)
6. [Understanding Partitioning in Azure SQL Database](#part-4-understanding-partitioning)
7. [E-commerce Partitioning Implementation](#part-5-ecommerce-partitioning-example)

---

## Architecture Overview

**What We're Building:**

```
┌────────────────────────────────────────────────────────────────┐
│           Azure Cloud Environment                              │
│           Resource Group: rg-ecommerce-demo-eastus-001        │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │   Windows Server 2022 VM                                │  │
│  │   Name: vm-dataserver-demo-001                          │  │
│  │   Size: Standard_D2s_v3 (2 vcpu, 8 GB)                 │  │
│  │   Public IP: pip-dataserver-demo-001                    │  │
│  │   OS Disk: disk-dataserver-os-001                       │  │
│  │                                                         │  │
│  │   ┌─────────────────────────────────────┐              │  │
│  │   │  Azure Data Studio Installed        │              │  │
│  │   │  - Query Editor                     │              │  │
│  │   │  - Database Management              │              │  │
│  │   └─────────────────────────────────────┘              │  │
│  │                                                         │  │
│  │   Network: vnet-ecommerce-demo-eastus-001              │  │
│  │   NSG: nsg-dataserver-demo-001                         │  │
│  └────────────────┬────────────────────────────────────────┘  │
│                   │                                           │
│                   │ SQL Connection (Port 1433)                │
│                   │ Encrypted (TLS)                           │
│                   ↓                                           │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │   Azure SQL Database (PaaS)                             │  │
│  │   Server: sql-ecommerce-demo-eastus-001                 │  │
│  │   Database: EcommerceDB                                 │  │
│  │   Tier: General Purpose                                 │  │
│  │   Service Level: S3 (100 DTUs)                          │  │
│  │                                                         │  │
│  │   Firewall Rules:                                       │  │
│  │   - AllowMyHomeIP (your IP)                            │  │
│  │   - Allow Azure services: Yes                           │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
         ↑
         │ RDP Connection (Port 3389)
         │ From your local computer
         │
    ┌────┴─────┐
    │   Your   │
    │ Computer │
    │          │
    └──────────┘
```

**Resource Naming Summary:**

| Resource Type | Resource Name | Purpose |
|--------------|---------------|---------|
| Resource Group | `rg-ecommerce-demo-eastus-001` | Contains all resources |
| SQL Server | `sql-ecommerce-demo-eastus-001` | Logical SQL Server |
| SQL Database | `EcommerceDB` | E-commerce database |
| Virtual Machine | `vm-dataserver-demo-001` | Windows Server for data work |
| Virtual Network | `vnet-ecommerce-demo-eastus-001` | Network for resources |
| Network Interface | `nic-dataserver-demo-001` | VM network interface |
| Network Security Group | `nsg-dataserver-demo-001` | Firewall rules for VM |
| Public IP | `pip-dataserver-demo-001` | Public IP for RDP |
| OS Disk | `disk-dataserver-os-001` | VM operating system disk |

**Key Components:**
1. **Azure SQL Database:** Managed database service (PaaS) - no server management required
2. **Windows Server VM:** Your work environment with Azure Data Studio
3. **Your Computer:** Connect to Windows Server via RDP
4. **Virtual Network:** Secure network isolating your resources
5. **Firewall Rules:** Control access to SQL Database

**Benefits of This Setup:**
- ✅ Managed database with automatic backups and updates
- ✅ Dedicated Windows Server environment for database work
- ✅ Can be accessed from anywhere via RDP
- ✅ No local installation needed on your computer
- ✅ Easy to scale both VM and database independently
- ✅ Consistent naming for easy resource management
- ✅ Cost optimization with auto-shutdown for VM

**Network Flow:**
1. You connect to `vm-dataserver-demo-001` via RDP (port 3389)
2. From VM, Azure Data Studio connects to `sql-ecommerce-demo-eastus-001` via SQL (port 1433)
3. Connection is encrypted with TLS
4. Firewall rules allow only authorized IP addresses

---

## Prerequisites

- Active Azure subscription
- Azure account with appropriate permissions
- Basic knowledge of SQL Server and T-SQL
- RDP client (Remote Desktop) for connecting to Windows Server
- Internet connection

---

## Azure Naming Conventions

### Why Naming Conventions Matter

Proper naming conventions help with:
- **Resource identification:** Quickly identify resource type and purpose
- **Organization:** Group related resources logically
- **Cost management:** Track spending by project or environment
- **Automation:** Easier scripting and infrastructure as code
- **Team collaboration:** Consistent understanding across teams

### Microsoft's Recommended Naming Convention

**Format:** `<resource-type>-<app-or-service>-<environment>-<region>-<instance>`

### Common Abbreviations for Azure Resources

| Resource Type | Abbreviation | Example |
|--------------|--------------|---------|
| Resource Group | rg | rg-ecommerce-prod-eastus-001 |
| SQL Server | sql | sql-ecommerce-prod-eastus-001 |
| SQL Database | sqldb | sqldb-ecommerce-prod-eastus-001 |
| Virtual Machine | vm | vm-dataserver-prod-eastus-001 |
| Virtual Network | vnet | vnet-ecommerce-prod-eastus-001 |
| Network Security Group | nsg | nsg-vm-prod-eastus-001 |
| Public IP Address | pip | pip-vm-prod-eastus-001 |
| Network Interface | nic | nic-vm-prod-eastus-001 |
| Storage Account | st | stecommercedata001 (no hyphens, lowercase) |
| Disk | disk | disk-vm-os-prod-001 |

### Environment Abbreviations

| Environment | Abbreviation |
|------------|--------------|
| Development | dev |
| Testing | test |
| Staging | stg |
| Production | prod |
| Demo | demo |

### Azure Region Abbreviations

| Region | Abbreviation |
|--------|--------------|
| East US | eastus |
| East US 2 | eastus2 |
| West US | westus |
| West US 2 | westus2 |
| Central US | centralus |
| North Europe | northeu |
| West Europe | westeu |
| Southeast Asia | southeastasia |
| UK South | uksouth |

### Naming Convention Examples for Our E-commerce Project

**Scenario:** E-commerce application, Demo environment, East US region

| Resource | Recommended Name | Why |
|----------|-----------------|-----|
| Resource Group | `rg-ecommerce-demo-eastus-001` | Groups all related resources |
| SQL Server | `sql-ecommerce-demo-eastus-001` | Identifies SQL server for demo |
| SQL Database | `sqldb-ecommerce-demo-eastus-001` | Specific database name |
| Windows VM | `vm-dataserver-demo-eastus-001` | VM for data operations |
| Virtual Network | `vnet-ecommerce-demo-eastus-001` | Network for the project |
| Network Security Group | `nsg-dataserver-demo-eastus-001` | Security group for VM |
| Public IP | `pip-dataserver-demo-eastus-001` | Public IP for RDP access |

### Important Naming Rules

#### SQL Server Name Rules:
- ✅ Lowercase letters, numbers, and hyphens
- ✅ 3-63 characters
- ✅ Must start with letter
- ✅ Must end with letter or number
- ✅ Must be globally unique (across all Azure)
- ❌ No underscores, special characters, or spaces
- **Example:** `sql-ecommerce-demo-eastus-001`

#### SQL Database Name Rules:
- ✅ Letters, numbers, hyphens, underscores
- ✅ 1-128 characters
- ✅ Cannot start or end with period
- ❌ No spaces or special characters
- **Example:** `EcommerceDB` or `sqldb-ecommerce-demo`

#### Virtual Machine Name Rules:
- ✅ Letters, numbers, and hyphens
- ✅ 1-15 characters (Windows)
- ✅ Must start with letter
- ✅ Must end with letter or number
- ❌ No underscores, special characters, or spaces
- **Example:** `vm-dataserver-001`

#### Storage Account Name Rules:
- ✅ Lowercase letters and numbers only
- ✅ 3-24 characters
- ✅ Must be globally unique
- ❌ No hyphens, underscores, or special characters
- **Example:** `stecommercedata001`

### Alternative Naming Approaches

#### Approach 1: Simplified (For Small Projects)
```
Resource Group:  rg-ecommerce
SQL Server:      sql-ecommerce
SQL Database:    EcommerceDB
VM:              vm-dataserver
```

#### Approach 2: Descriptive (For Large Organizations)
```
Resource Group:  rg-retail-ecommerce-demo-eastus-001
SQL Server:      sql-retail-ecommerce-demo-eastus-001
SQL Database:    sqldb-retail-ecommerce-orders-demo
VM:              vm-retail-data-analytics-demo-001
```

#### Approach 3: Project Code Based
```
Resource Group:  rg-prj12345-demo-eastus
SQL Server:      sql-prj12345-demo-eastus
SQL Database:    sqldb-prj12345-orders
VM:              vm-prj12345-data-001
```

### Tagging Strategy (Complement to Naming)

In addition to naming, use Azure Tags:

| Tag Name | Example Value | Purpose |
|----------|---------------|---------|
| Environment | Demo, Dev, Prod | Identify environment |
| Project | Ecommerce | Project name |
| CostCenter | IT-Database | Billing tracking |
| Owner | john.doe@company.com | Resource owner |
| Application | Ecommerce Platform | Application name |
| Criticality | High, Medium, Low | Business criticality |

**How to add tags in Azure Portal:**
1. Navigate to any resource
2. Click "Tags" in the left menu
3. Add name-value pairs
4. Click "Apply"

### Best Practices Summary

1. ✅ **Be consistent** across all resources
2. ✅ **Use lowercase** for most resources
3. ✅ **Include environment** (dev, test, prod)
4. ✅ **Include region** for multi-region deployments
5. ✅ **Add instance number** (001, 002) for scaling
6. ✅ **Keep it readable** - balance between descriptive and concise
7. ✅ **Document your convention** - share with team
8. ✅ **Use tags** to complement naming
9. ✅ **Check uniqueness** for globally unique resources
10. ✅ **Plan for growth** - your naming should scale

### Quick Reference: Our Tutorial Naming

For this tutorial, we'll use:

```
Resource Group:       rg-ecommerce-demo-eastus-001
SQL Server:          sql-ecommerce-demo-eastus-001
SQL Database:        EcommerceDB
Windows VM:          vm-dataserver-demo-001
Virtual Network:     vnet-ecommerce-demo-eastus-001
Network Interface:   nic-dataserver-demo-001
Public IP:           pip-dataserver-demo-001
NSG:                 nsg-dataserver-demo-001
OS Disk:             disk-dataserver-os-001
```

---

## Part 1: Creating Azure SQL Database Server

### Step 1: Sign in to Azure Portal

1. Navigate to https://portal.azure.com
2. Sign in with your Azure credentials

### Step 2: Create SQL Database Server

1. **Navigate to SQL Databases**
   - Click on "Create a resource"
   - Search for "SQL Database"
   - Click "Create"

2. **Configure Basic Settings**

   **Project Details:**
   - Subscription: Select your subscription
   - Resource Group: Click "Create new"
     - Name: `rg-ecommerce-demo-eastus-001`
     - Click "OK"
   
   **Database Details:**
   - Database name: `EcommerceDB`
   - Server: Click "Create new"

3. **Create New SQL Server**

   **Server Details:**
   - Server name: `sql-ecommerce-demo-eastus-001`
     - Note: This must be globally unique. If taken, try:
       - `sql-ecommerce-demo-eastus-002`
       - `sql-ecommerce-yourname-eastus-001`
       - `sql-ecommerce-demo-[random-number]`
     - The full server address will be: `sql-ecommerce-demo-eastus-001.database.windows.net`
   - Location: **East US** (or your preferred region)
   - Authentication method: **Use SQL authentication**
   
   **Server Admin Login:**
   - Server admin login: `sqladmin`
   - Password: Create a strong password (min 12 characters)
     - Must contain: uppercase, lowercase, numbers, special characters
     - Example format: `P@ssw0rd2024!Demo`
   - Confirm password: Re-enter password
   
   **IMPORTANT:** Save these credentials securely!
   ```
   Server: sql-ecommerce-demo-eastus-001.database.windows.net
   Username: sqladmin
   Password: [your password]
   ```
   
   - Click "OK"

4. **Compute + Storage Configuration**

   - Click "Configure database"
   - Service tier: **General Purpose**
   - Compute tier: **Provisioned**
   - Service level: **S3 (100 DTUs)** or higher for better performance
   - Click "Apply"

5. **Backup Storage Redundancy**
   - Select "Locally-redundant backup storage" (for demo)

6. **Networking Configuration**

   - Click "Next: Networking"
   - Connectivity method: **Public endpoint**
   
   **Firewall rules:**
   - Allow Azure services: **Yes**
   - Add current client IP address: **Yes** (important!)
   
   - Click "Next: Security"

7. **Security Settings**
   - Enable Microsoft Defender for SQL: No (optional, for demo)
   - Click "Next: Additional settings"

8. **Additional Settings**
   - Use existing data: **None**
   - Collation: **Default**
   - Click "Review + create"

9. **Review and Create**
   - Review all settings
   - Click "Create"
   - Wait 3-5 minutes for deployment

### Step 3: Configure Firewall Rules (If Needed)

1. **Navigate to SQL Server**
   - Go to your resource group: `rg-ecommerce-demo-eastus-001`
   - Click on your SQL server: `sql-ecommerce-demo-eastus-001`

2. **Add Firewall Rule**
   - In left menu, click "Networking" or "Firewalls and virtual networks"
   - Under "Firewall rules", click "+ Add a firewall rule"
   - Rule name: `AllowMyHomeIP` or `AllowMyOfficeIP`
   - Start IP: Your public IP address
   - End IP: Your public IP address (same as start for single IP)
   - Click "Save"

   > **Tip:** To find your public IP, search "what is my ip" in Google
   > **Alternative:** You can use a range like `AllowOfficeRange` with start IP 203.0.113.1 and end IP 203.0.113.254

3. **Verify Azure Services Access**
   - Ensure "Allow Azure services and resources to access this server" is set to **Yes**
   - This allows your Azure VM to connect to the database
   - Click "Save" if you made changes

### Step 4: Get Connection Information

1. **Copy Connection String**
   - Navigate to your database `EcommerceDB`
   - Click "Connection strings" in left menu
   - Copy the **ADO.NET** connection string
   - Save it in Notepad for reference
   
   **Example Connection String:**
   ```
   Server=tcp:sql-ecommerce-demo-eastus-001.database.windows.net,1433;
   Initial Catalog=EcommerceDB;
   Persist Security Info=False;
   User ID=sqladmin;
   Password={your_password_here};
   MultipleActiveResultSets=False;
   Encrypt=True;
   TrustServerCertificate=False;
   Connection Timeout=30;
   ```

2. **Key Connection Parameters**
   - **Server:** `sql-ecommerce-demo-eastus-001.database.windows.net`
   - **Port:** `1433` (default SQL Server port)
   - **Database:** `EcommerceDB`
   - **Username:** `sqladmin`
   - **Password:** Your password from setup
   - **Encryption:** Mandatory (True)

3. **Save Credentials Securely**
   Create a text file with your connection details:
   ```
   Azure SQL Server Connection Details
   ====================================
   Server Name: sql-ecommerce-demo-eastus-001.database.windows.net
   Database: EcommerceDB
   Username: sqladmin
   Password: [your_password]
   Port: 1433
   Resource Group: rg-ecommerce-demo-eastus-001
   Region: East US
   Created: [date]
   ```
   
   Save this file securely (e.g., in a password manager or encrypted location)

---

## Part 2: Creating Windows Server VM and Installing Azure Data Studio

### Step 1: Create Windows Server Virtual Machine in Azure

1. **Navigate to Virtual Machines**
   - In Azure Portal, click "Create a resource"
   - Search for "Virtual Machine"
   - Click "Create" → "Virtual Machine"

2. **Configure Basic Settings**

   **Project Details:**
   - Subscription: Select your subscription
   - Resource Group: **Use existing** → `rg-ecommerce-demo-eastus-001`
     - (We created this with the SQL Server)
   
   **Instance Details:**
   - Virtual machine name: `vm-dataserver-demo-001`
     - Follows naming convention: vm-[purpose]-[environment]-[instance]
   - Region: **East US** (same as your SQL Database for best performance)
   - Availability options: No infrastructure redundancy required
   - Security type: Standard
   - Image: **Windows Server 2022 Datacenter: Azure Edition - x64 Gen2**
   - Size: **Standard_D2s_v3** (2 vcpus, 8 GiB memory)
     - Good balance for data work
     - Click "See all sizes" for other options:
       - Standard_B2s (2 vcpu, 4 GB) - Cheaper, good for testing
       - Standard_D4s_v3 (4 vcpu, 16 GB) - Better for heavy workloads
   
   **Administrator Account:**
   - Username: `azureadmin`
     - Avoid using "admin" or "administrator" (blocked by Azure)
   - Password: Create a strong password (minimum 12 characters)
     - Must contain: uppercase, lowercase, number, special character
     - Example format: `Azur3Adm!n2024`
   - Confirm password: Re-enter password
   - **IMPORTANT:** Save these credentials!
   ```
   VM Name: vm-dataserver-demo-001
   Username: azureadmin
   Password: [your_password]
   Public IP: [will be shown after creation]
   ```

   **Inbound Port Rules:**
   - Public inbound ports: **Allow selected ports**
   - Select inbound ports: **RDP (3389)**
     - This allows Remote Desktop connection

3. **Configure Disks**
   - Click "Next: Disks"
   
   **OS Disk:**
   - OS disk type: **Premium SSD** (recommended for better performance)
     - Premium SSD: Best performance, higher cost
     - Standard SSD: Good balance
     - Standard HDD: Lowest cost, slower
   - Encryption type: **(Default) Encryption at rest with platform-managed key**
   - Delete with VM: ✓ Checked (recommended for demos)
   
   **Data Disks:**
   - We don't need additional data disks for this tutorial
   - Click "Next: Networking"

4. **Configure Networking**
   
   **Network Interface:**
   - Virtual network: 
     - If exists: Use existing `vnet-ecommerce-demo-eastus-001`
     - If not: Click "Create new"
       - Name: `vnet-ecommerce-demo-eastus-001`
       - Address space: 10.0.0.0/16
       - Click "OK"
   
   - Subnet: 
     - Name: `snet-dataserver-demo-001` (or default)
     - Address range: 10.0.0.0/24
   
   - Public IP: **Create new**
     - Click on "(new) vm-dataserver-demo-001-ip"
     - Change name to: `pip-dataserver-demo-001`
     - SKU: Standard
     - Assignment: Static (recommended) or Dynamic
     - Click "OK"
   
   - NIC network security group: **Basic**
   - Public inbound ports: **Allow selected ports**
   - Select inbound ports: **RDP (3389)**
   
   - Delete public IP and NIC when VM is deleted: ✓ Checked
   
   **Advanced Settings:**
   - Accelerated networking: Keep default (usually enabled)
   
   - Click "Next: Management"

5. **Management Settings**
   
   **Monitoring:**
   - Boot diagnostics: Enabled (recommended)
   - OS guest diagnostics: Optional
   
   **Auto-shutdown (Highly Recommended for Cost Saving):**
   - Enable auto-shutdown: ✓ **Yes**
   - Shutdown time: Select time (e.g., 7:00 PM or 19:00)
   - Time zone: Select your timezone
   - Notification before shutdown: ✓ Yes
   - Email address: Your email
   - Webhook URL: Leave blank
   
   This will automatically shut down the VM daily to save costs!
   
   **Identity:**
   - System assigned managed identity: Off (not needed for this tutorial)
   
   - Click "Next: Advanced" (or skip to "Review + create")

6. **Review and Create**
   - Review all settings
   - Verify naming conventions:
     - VM name: `vm-dataserver-demo-001`
     - Resource group: `rg-ecommerce-demo-eastus-001`
     - Public IP: `pip-dataserver-demo-001`
     - Virtual network: `vnet-ecommerce-demo-eastus-001`
   
   - Check estimated monthly cost
     - Standard_D2s_v3 costs approximately $70-100/month
     - With auto-shutdown (12 hours/day), cost reduces by ~50%
   
   - Click "Create"
   - Wait 3-5 minutes for deployment
   
   **Deployment Progress:**
   - You'll see resources being created:
     - Virtual machine
     - Network interface (nic-dataserver-demo-001)
     - Network security group (nsg-dataserver-demo-001)
     - Public IP address (pip-dataserver-demo-001)
     - OS disk (disk-dataserver-os-001)

7. **Post-Deployment**
   - Once deployment completes, click "Go to resource"
   - You'll see the VM overview page
   - **Note the Public IP address** - you'll need this for RDP connection
   
   **Save VM Details:**
   ```
   VM Name: vm-dataserver-demo-001
   Resource Group: rg-ecommerce-demo-eastus-001
   Region: East US
   Size: Standard_D2s_v3
   Public IP: [shown in overview - e.g., 20.185.123.45]
   Username: azureadmin
   Password: [your_password]
   RDP Port: 3389
   Auto-shutdown: Enabled at 7:00 PM
   ```

### Step 2: Connect to Windows Server VM

1. **Get Connection Details**
   - Once deployment is complete, click "Go to resource"
   - On the VM overview page, note the **Public IP address**
   - Click "Connect" → "RDP"
   - Click "Download RDP File"

2. **Connect via RDP**
   - Locate and open the downloaded `.rdp` file
   - Click "Connect"
   - Enter credentials:
     - Username: `azureadmin` (or your username)
     - Password: Your password from setup
   - Click "OK"
   
3. **Certificate Warning**
   - You may see a certificate warning
   - Click "Yes" or "Connect anyway"
   - You're now connected to Windows Server!

### Step 3: Configure Windows Server

1. **Disable IE Enhanced Security (for downloading)**
   - Server Manager should open automatically
   - If not, click Start → Server Manager
   - Click "Local Server" in the left panel
   - Find "IE Enhanced Security Configuration"
   - Click "On"
   - Set both Administrators and Users to "Off"
   - Click "OK"

2. **Optional: Disable Server Manager Auto-Start**
   - In Server Manager, click "Manage" → "Server Manager Properties"
   - Check "Do not start Server Manager automatically at logon"
   - Click "OK"

### Step 4: Install Azure Data Studio on Windows Server

1. **Download Azure Data Studio**
   - Open Microsoft Edge (pre-installed on Windows Server)
   - Navigate to: https://docs.microsoft.com/en-us/sql/azure-data-studio/download
   - Click "**Windows System Installer (recommended)**"
   - Save the file to Downloads folder
   - Wait for download to complete

2. **Run the Installer**
   - Open File Explorer
   - Navigate to Downloads folder
   - Double-click the installer file:
     - `azuredatastudio-windows-setup-<version>.exe`
   
3. **Installation Wizard Steps**
   
   **License Agreement:**
   - Read and accept the license agreement
   - Click "Next"
   
   **Select Destination Location:**
   - Default location: `C:\Program Files\Azure Data Studio`
   - Click "Next"
   
   **Select Start Menu Folder:**
   - Default: "Azure Data Studio"
   - Click "Next"
   
   **Select Additional Tasks:**
   - ✓ Create a desktop icon (recommended)
   - ✓ Add to PATH (recommended)
   - ✓ Register Code as an editor for supported file types
   - ✓ Add "Open with Code" action to Windows Explorer file context menu
   - ✓ Add "Open with Code" action to Windows Explorer directory context menu
   - Click "Next"
   
   **Ready to Install:**
   - Review your choices
   - Click "Install"
   - Wait for installation (2-3 minutes)
   
   **Completing Setup:**
   - ✓ Launch Azure Data Studio (keep this checked)
   - Click "Finish"

4. **First Launch Configuration**
   - Azure Data Studio will open
   - You may see a Windows Firewall prompt
   - Click "Allow access" to enable network features
   
5. **Verify Installation**
   - Azure Data Studio should now be running
   - You should see the Welcome page
   - Click Help → About to verify version

### Step 5: Install Additional Tools (Optional but Recommended)

1. **Install SQL Server Management Studio (SSMS) - Optional**
   ```
   Navigate to: https://aka.ms/ssmsfullsetup
   Download and install if you want additional SQL Server tools
   ```

2. **Install .NET Framework (if needed)**
   - Usually pre-installed on Windows Server 2022
   - If prompted during Azure Data Studio installation, install it

3. **Install Git (Optional - for version control)**
   - Navigate to: https://git-scm.com/download/win
   - Download and install Git for Windows

### Step 6: Configure Windows Server for Better Performance

1. **Adjust Visual Effects for Better Performance**
   - Right-click "This PC" → Properties
   - Click "Advanced system settings"
   - Under Performance, click "Settings"
   - Select "Adjust for best performance"
   - Or keep "Custom" and uncheck unnecessary effects
   - Click "Apply" → "OK"

2. **Set Power Plan to High Performance**
   - Open Control Panel
   - Go to "Power Options"
   - Select "High performance"

3. **Configure Automatic Updates**
   - Open Settings → Update & Security
   - Configure as per your organization policy
   - For testing: You can defer updates

### Step 7: Create Desktop Shortcuts

1. **Pin Azure Data Studio to Taskbar**
   - Right-click Azure Data Studio icon on desktop
   - Select "Pin to taskbar"

2. **Create Quick Access Folder for Scripts**
   - Create folder: `C:\SQLScripts`
   - Right-click → Pin to Quick access in File Explorer

### Troubleshooting Windows Server Setup

**Issue 1: Cannot Download Files**
- Problem: Internet Explorer blocks downloads
- Solution: Disable IE Enhanced Security Configuration (see Step 3.1)

**Issue 2: RDP Connection Fails**
- Problem: Cannot connect to VM
- Solution: 
  - Verify VM is running in Azure Portal
  - Check Network Security Group allows RDP (port 3389)
  - Verify your public IP is allowed in NSG rules

**Issue 3: Slow Performance**
- Problem: VM is slow
- Solution:
  - Ensure you selected appropriate VM size
  - Close unnecessary applications
  - Adjust visual effects (see Step 6.1)

**Issue 4: Azure Data Studio Won't Install**
- Problem: Installation fails
- Solution:
  - Run installer as Administrator
  - Check if .NET Framework is installed
  - Temporarily disable Windows Defender

### Windows Server VM Cost Optimization

1. **Auto-Shutdown**
   - Enable auto-shutdown in Azure Portal
   - Set time when you're not working (e.g., 7 PM)
   - This saves significant costs!

2. **Deallocate When Not in Use**
   - In Azure Portal, click "Stop" on your VM
   - This deallocates and stops billing for compute
   - Storage costs still apply but are minimal

3. **Right-Size Your VM**
   - Start with Standard_D2s_v3
   - Scale up only if needed
   - Scale down when testing is complete

### Next Steps

Now that you have Windows Server VM with Azure Data Studio installed, you can proceed to Part 3 to connect to your Azure SQL Database!

---

## Part 3: Connecting to Azure SQL Database from Windows Server

### Step 1: Launch Azure Data Studio on Windows Server

1. **Open Azure Data Studio**
   - Double-click the Azure Data Studio icon on desktop
   - Or click Start → Azure Data Studio
   - Or use taskbar shortcut if pinned

### Step 2: Create New Connection

1. **Open Connection Dialog**
   - Click "New Connection" button on the Welcome page
   - Or click the "Connections" icon in the left sidebar
   - Or press `Ctrl+Shift+C`

2. **Enter Connection Details**

   - **Connection type:** `Microsoft SQL Server`
   - **Server:** `sql-ecommerce-demo-eastus-001.database.windows.net`
     - Use the full server name from Part 1
     - Must include `.database.windows.net`
   - **Authentication type:** `SQL Login`
   - **User name:** `sqladmin` (or your admin username)
   - **Password:** Your password from SQL Server setup
   - **Remember password:** ✓ Checked (recommended)
   - **Database:** `EcommerceDB`
     - Or select `<Default>` to see all databases
   - **Encrypt:** `Mandatory (True)` - **REQUIRED for Azure SQL Database**
   - **Trust server certificate:** `False`
   - **Server group:** `<Default>` or create new
     - To organize: Create group named "Azure SQL Servers"
   - **Name (optional):** `Azure Ecommerce DB - Demo`
     - Friendly name to identify this connection

   **Example Connection Configuration:**
   ```
   Connection Type: Microsoft SQL Server
   Server: sql-ecommerce-demo-eastus-001.database.windows.net
   Authentication: SQL Login
   User name: sqladmin
   Password: ••••••••••
   Database: EcommerceDB
   Encrypt: Mandatory (True)
   Name: Azure Ecommerce DB - Demo
   ```

3. **Test and Connect**
   - Click "Connect"
   - If successful, you'll see:
     - Server name in the left "CONNECTIONS" panel
     - Database "EcommerceDB" underneath
     - Green dot indicating active connection
   
   **First Connection Success:**
   ```
   ✓ Connected to: sql-ecommerce-demo-eastus-001.database.windows.net
   ✓ Database: EcommerceDB
   ✓ Authentication: Successful
   ✓ Encryption: Enabled
   ```

4. **Verify Connection**
   - Right-click on `EcommerceDB` in Connections panel
   - Select "New Query"
   - Run a test query:
   ```sql
   SELECT 
       @@VERSION AS SQLVersion,
       DB_NAME() AS DatabaseName,
       SUSER_SNAME() AS LoginName,
       GETDATE() AS CurrentDateTime;
   ```
   - Press F5 or click "Run"
   - You should see results showing SQL Server version and database details

### Troubleshooting Connection Issues

**Issue 1: Firewall Block**
- Error: "Cannot open server... Client with IP address 'x.x.x.x' is not allowed"
- **Solution:** 
  1. Go to Azure Portal
  2. Navigate to `sql-ecommerce-demo-eastus-001` SQL Server
  3. Click "Networking" or "Firewalls and virtual networks"
  4. Add firewall rule:
     - Name: `AllowVMIP` or `AllowMyCurrentIP`
     - Start IP: Your current public IP (or VM's public IP)
     - End IP: Same as start IP
  5. Ensure "Allow Azure services" is set to **Yes**
  6. Click "Save" and wait 2-3 minutes
  7. Try connecting again from Azure Data Studio

**Issue 2: Invalid Credentials**
- Error: "Login failed for user 'sqladmin'"
- **Solution:** 
  1. Verify username is exactly: `sqladmin`
  2. Check password - it's case-sensitive
  3. Reset password if needed:
     - Go to Azure Portal
     - Navigate to SQL Server: `sql-ecommerce-demo-eastus-001`
     - Click "Reset password"
     - Enter new password
     - Click "Save"

**Issue 3: Server Name Error**
- Error: "A network-related or instance-specific error occurred"
- **Solution:** 
  - Ensure server name is complete: `sql-ecommerce-demo-eastus-001.database.windows.net`
  - Common mistakes:
    - ❌ `sql-ecommerce-demo-eastus-001` (missing .database.windows.net)
    - ❌ `sql-ecommerce-demo-eastus-001.database.windows.com` (wrong domain)
    - ✅ `sql-ecommerce-demo-eastus-001.database.windows.net` (correct)

**Issue 4: Encryption Error**
- Error: "A connection was successfully established... but an error occurred during the login"
- **Solution:** 
  - In Azure Data Studio connection settings:
  - Set "Encrypt" to **Mandatory (True)**
  - Azure SQL Database REQUIRES encryption

**Issue 5: Database Not Found**
- Error: "Cannot open database 'EcommerceDB'"
- **Solution:** 
  1. Verify database exists in Azure Portal
  2. Check database name spelling (case-sensitive)
  3. Try connecting with `<Default>` database first
  4. Then select EcommerceDB from dropdown

**Issue 6: Timeout**
- Error: "Connection timeout expired"
- **Solution:** 
  - Check your internet connection
  - Verify VM can reach internet (ping 8.8.8.8)
  - Increase connection timeout in Azure Data Studio:
    - Advanced connection settings
    - Connection Timeout: 60 seconds

**Verify Your Resource Names:**
If you're having issues, verify all your resources follow the naming convention:
```
Resource Group:  rg-ecommerce-demo-eastus-001
SQL Server:      sql-ecommerce-demo-eastus-001
SQL Database:    EcommerceDB
Windows VM:      vm-dataserver-demo-001
Public IP:       pip-dataserver-demo-001
```

Run this query in Azure Data Studio to verify connection:
```sql
SELECT 
    @@SERVERNAME AS ServerName,
    DB_NAME() AS DatabaseName,
    SYSTEM_USER AS LoginUser,
    @@VERSION AS SQLVersion;
```

---

## Part 4: Understanding Partitioning in Azure SQL Database

### Key Differences from On-Premises SQL Server

**Azure SQL Database Limitations:**
1. **No Custom Filegroups:** You cannot create custom filegroups
2. **PRIMARY Filegroup Only:** All partitions use PRIMARY filegroup
3. **No Physical File Control:** Cannot specify file locations or sizes
4. **Simplified Management:** Azure handles physical storage

**What Still Works:**
- ✅ Partition Functions
- ✅ Partition Schemes (but mapped to PRIMARY only)
- ✅ Partitioned Tables
- ✅ Partition Elimination
- ✅ Sliding Window scenarios
- ✅ All query benefits

### Partitioning Benefits in Azure SQL Database

1. **Query Performance:** Partition elimination reduces I/O
2. **Data Management:** Easier to archive/load data
3. **Maintenance:** Index maintenance on specific partitions
4. **Scalability:** Better handling of large tables
5. **Cost Optimization:** Efficient resource utilization

---

## Part 5: E-commerce Partitioning Example

### Scenario

We'll create an e-commerce database with:
- **Orders Table:** Partitioned by OrderDate (yearly partitions)
- **OrderItems Table:** Partitioned by OrderDate (aligned with Orders)
- **CustomerActivity Table:** Partitioned by ActivityDate (monthly partitions)
- **Customers & Products:** Non-partitioned reference tables

### Step 1: Create New Query in Azure Data Studio

1. In Azure Data Studio, click "New Query"
2. Ensure you're connected to `EcommerceDB`
3. Copy and execute the following scripts

### Step 2: Create Partition Function for Orders (Yearly)

```sql
-- ============================================
-- E-COMMERCE DATABASE PARTITIONING SETUP
-- ============================================

-- Create partition function for yearly order partitioning
CREATE PARTITION FUNCTION PF_OrdersByYear (DATETIME2)
AS RANGE RIGHT FOR VALUES 
(
    '2022-01-01',  -- Partition 1: < 2022-01-01 (historical)
    '2023-01-01',  -- Partition 2: 2022 data
    '2024-01-01',  -- Partition 3: 2023 data
    '2025-01-01',  -- Partition 4: 2024 data
    '2026-01-01'   -- Partition 5: 2025 data
                   -- Partition 6: >= 2026-01-01 (future)
);
GO

-- Verify partition function
SELECT 
    pf.name AS PartitionFunction,
    pf.type_desc AS RangeType,
    pf.fanout AS NumberOfPartitions,
    prv.boundary_id,
    prv.value AS BoundaryValue
FROM sys.partition_functions pf
LEFT JOIN sys.partition_range_values prv ON pf.function_id = prv.function_id
WHERE pf.name = 'PF_OrdersByYear'
ORDER BY prv.boundary_id;
GO
```

### Step 3: Create Partition Scheme

```sql
-- Create partition scheme (all partitions on PRIMARY in Azure SQL DB)
CREATE PARTITION SCHEME PS_OrdersByYear
AS PARTITION PF_OrdersByYear
ALL TO ([PRIMARY]);
GO

-- Verify partition scheme
SELECT 
    ps.name AS PartitionScheme,
    pf.name AS PartitionFunction,
    dds.destination_id AS PartitionNumber,
    ds.name AS FileGroupName
FROM sys.partition_schemes ps
INNER JOIN sys.partition_functions pf ON ps.function_id = pf.function_id
INNER JOIN sys.destination_data_spaces dds ON ps.data_space_id = dds.partition_scheme_id
INNER JOIN sys.data_spaces ds ON dds.data_space_id = ds.data_space_id
WHERE ps.name = 'PS_OrdersByYear'
ORDER BY dds.destination_id;
GO
```

### Step 4: Create Base Tables

```sql
-- ============================================
-- CREATE BASE TABLES (NON-PARTITIONED)
-- ============================================

-- Customers table
CREATE TABLE Customers
(
    CustomerID INT IDENTITY(1,1) PRIMARY KEY CLUSTERED,
    FirstName NVARCHAR(50) NOT NULL,
    LastName NVARCHAR(50) NOT NULL,
    Email NVARCHAR(100) NOT NULL UNIQUE,
    Phone NVARCHAR(20),
    Address NVARCHAR(200),
    City NVARCHAR(50),
    State NVARCHAR(50),
    ZipCode NVARCHAR(10),
    Country NVARCHAR(50),
    DateOfBirth DATE,
    CreatedDate DATETIME2 DEFAULT GETDATE(),
    LastModifiedDate DATETIME2 DEFAULT GETDATE()
);
GO

-- Products table
CREATE TABLE Products
(
    ProductID INT IDENTITY(1,1) PRIMARY KEY CLUSTERED,
    ProductName NVARCHAR(100) NOT NULL,
    SKU NVARCHAR(50) UNIQUE NOT NULL,
    CategoryID INT,
    Price DECIMAL(10,2) NOT NULL,
    Cost DECIMAL(10,2),
    StockQuantity INT NOT NULL DEFAULT 0,
    Description NVARCHAR(500),
    IsActive BIT DEFAULT 1,
    CreatedDate DATETIME2 DEFAULT GETDATE(),
    LastModifiedDate DATETIME2 DEFAULT GETDATE()
);
GO

-- Categories table
CREATE TABLE Categories
(
    CategoryID INT IDENTITY(1,1) PRIMARY KEY CLUSTERED,
    CategoryName NVARCHAR(50) NOT NULL,
    Description NVARCHAR(200),
    ParentCategoryID INT NULL,
    CONSTRAINT FK_Categories_Parent FOREIGN KEY (ParentCategoryID) 
        REFERENCES Categories(CategoryID)
);
GO

-- Add foreign key to Products
ALTER TABLE Products
ADD CONSTRAINT FK_Products_Categories 
    FOREIGN KEY (CategoryID) REFERENCES Categories(CategoryID);
GO
```

### Step 5: Create Partitioned Orders Table

```sql
-- ============================================
-- CREATE PARTITIONED ORDERS TABLE
-- ============================================

CREATE TABLE Orders
(
    OrderID INT IDENTITY(1,1),
    CustomerID INT NOT NULL,
    OrderDate DATETIME2 NOT NULL DEFAULT GETDATE(),
    TotalAmount DECIMAL(10,2) NOT NULL,
    OrderStatus NVARCHAR(20) NOT NULL 
        CONSTRAINT CHK_OrderStatus CHECK (OrderStatus IN ('Pending', 'Processing', 'Shipped', 'Delivered', 'Cancelled', 'Completed')),
    ShippingAddress NVARCHAR(200),
    BillingAddress NVARCHAR(200),
    PaymentMethod NVARCHAR(50),
    PaymentStatus NVARCHAR(20) DEFAULT 'Pending',
    ShippingMethod NVARCHAR(50),
    TrackingNumber NVARCHAR(100),
    Notes NVARCHAR(500),
    CreatedDate DATETIME2 DEFAULT GETDATE(),
    LastModifiedDate DATETIME2 DEFAULT GETDATE(),
    
    -- Partitioned primary key must include partition column
    CONSTRAINT PK_Orders PRIMARY KEY CLUSTERED (OrderID, OrderDate),
    
    -- Foreign key constraint
    CONSTRAINT FK_Orders_Customers FOREIGN KEY (CustomerID) 
        REFERENCES Customers(CustomerID)
        
) ON PS_OrdersByYear(OrderDate);  -- Partition on OrderDate
GO

-- Create non-clustered indexes on Orders
CREATE NONCLUSTERED INDEX IX_Orders_CustomerID 
    ON Orders(CustomerID, OrderDate)
    ON PS_OrdersByYear(OrderDate);
GO

CREATE NONCLUSTERED INDEX IX_Orders_OrderStatus 
    ON Orders(OrderStatus, OrderDate)
    ON PS_OrdersByYear(OrderDate);
GO

CREATE NONCLUSTERED INDEX IX_Orders_PaymentStatus 
    ON Orders(PaymentStatus, OrderDate)
    ON PS_OrdersByYear(OrderDate);
GO
```

### Step 6: Create Partitioned OrderItems Table

```sql
-- ============================================
-- CREATE PARTITIONED ORDER ITEMS TABLE
-- ============================================

CREATE TABLE OrderItems
(
    OrderItemID INT IDENTITY(1,1),
    OrderID INT NOT NULL,
    OrderDate DATETIME2 NOT NULL,  -- Must match Orders partition column
    ProductID INT NOT NULL,
    Quantity INT NOT NULL CHECK (Quantity > 0),
    UnitPrice DECIMAL(10,2) NOT NULL,
    Discount DECIMAL(5,2) DEFAULT 0,
    TaxAmount DECIMAL(10,2) DEFAULT 0,
    Subtotal AS (Quantity * UnitPrice * (1 - Discount/100) + TaxAmount) PERSISTED,
    
    -- Partitioned primary key
    CONSTRAINT PK_OrderItems PRIMARY KEY CLUSTERED (OrderItemID, OrderDate),
    
    -- Foreign keys
    CONSTRAINT FK_OrderItems_Orders FOREIGN KEY (OrderID, OrderDate) 
        REFERENCES Orders(OrderID, OrderDate),
    CONSTRAINT FK_OrderItems_Products FOREIGN KEY (ProductID) 
        REFERENCES Products(ProductID)
        
) ON PS_OrdersByYear(OrderDate);  -- Aligned partition
GO

-- Create indexes on OrderItems
CREATE NONCLUSTERED INDEX IX_OrderItems_OrderID 
    ON OrderItems(OrderID, OrderDate)
    ON PS_OrdersByYear(OrderDate);
GO

CREATE NONCLUSTERED INDEX IX_OrderItems_ProductID 
    ON OrderItems(ProductID, OrderDate)
    ON PS_OrdersByYear(OrderDate);
GO
```

### Step 7: Create Monthly Partitioned Activity Log

```sql
-- ============================================
-- CREATE MONTHLY PARTITIONED ACTIVITY LOG
-- ============================================

-- Partition function for monthly customer activity
CREATE PARTITION FUNCTION PF_ActivityByMonth (DATETIME2)
AS RANGE RIGHT FOR VALUES 
(
    '2024-01-01', '2024-02-01', '2024-03-01', '2024-04-01', '2024-05-01', '2024-06-01',
    '2024-07-01', '2024-08-01', '2024-09-01', '2024-10-01', '2024-11-01', '2024-12-01',
    '2025-01-01', '2025-02-01', '2025-03-01', '2025-04-01', '2025-05-01', '2025-06-01',
    '2025-07-01', '2025-08-01', '2025-09-01', '2025-10-01', '2025-11-01', '2025-12-01',
    '2026-01-01'
);
GO

-- Partition scheme for activity
CREATE PARTITION SCHEME PS_ActivityByMonth
AS PARTITION PF_ActivityByMonth
ALL TO ([PRIMARY]);
GO

-- Customer activity log table
CREATE TABLE CustomerActivity
(
    ActivityID BIGINT IDENTITY(1,1),
    CustomerID INT NOT NULL,
    ActivityDate DATETIME2 NOT NULL DEFAULT GETDATE(),
    ActivityType NVARCHAR(50) NOT NULL,  -- Login, View, AddToCart, Purchase, etc.
    PageURL NVARCHAR(500),
    ProductID INT NULL,
    SessionID NVARCHAR(100),
    IPAddress NVARCHAR(50),
    UserAgent NVARCHAR(500),
    
    CONSTRAINT PK_CustomerActivity PRIMARY KEY CLUSTERED (ActivityID, ActivityDate),
    CONSTRAINT FK_CustomerActivity_Customers FOREIGN KEY (CustomerID) 
        REFERENCES Customers(CustomerID)
        
) ON PS_ActivityByMonth(ActivityDate);
GO

CREATE NONCLUSTERED INDEX IX_CustomerActivity_CustomerID 
    ON CustomerActivity(CustomerID, ActivityDate)
    ON PS_ActivityByMonth(ActivityDate);
GO
```

### Step 8: Insert Sample Data

```sql
-- ============================================
-- INSERT SAMPLE DATA
-- ============================================

-- Insert Categories
INSERT INTO Categories (CategoryName, Description)
VALUES 
('Electronics', 'Electronic devices and accessories'),
('Audio', 'Audio equipment and accessories'),
('Office', 'Office supplies and furniture'),
('Computing', 'Computers and computer accessories');
GO

-- Insert Customers
INSERT INTO Customers (FirstName, LastName, Email, Phone, Address, City, State, ZipCode, Country, DateOfBirth)
VALUES 
('John', 'Doe', 'john.doe@email.com', '555-0001', '123 Main St', 'New York', 'NY', '10001', 'USA', '1985-05-15'),
('Jane', 'Smith', 'jane.smith@email.com', '555-0002', '456 Oak Ave', 'Los Angeles', 'CA', '90001', 'USA', '1990-08-22'),
('Robert', 'Johnson', 'robert.j@email.com', '555-0003', '789 Pine Rd', 'Chicago', 'IL', '60601', 'USA', '1988-03-10'),
('Emily', 'Brown', 'emily.brown@email.com', '555-0004', '321 Elm St', 'Houston', 'TX', '77001', 'USA', '1992-11-30'),
('Michael', 'Davis', 'michael.d@email.com', '555-0005', '654 Maple Dr', 'Phoenix', 'AZ', '85001', 'USA', '1987-07-18'),
('Sarah', 'Wilson', 'sarah.w@email.com', '555-0006', '987 Cedar Ln', 'Philadelphia', 'PA', '19019', 'USA', '1991-02-25'),
('David', 'Martinez', 'david.m@email.com', '555-0007', '147 Birch St', 'San Antonio', 'TX', '78201', 'USA', '1989-09-14'),
('Lisa', 'Anderson', 'lisa.a@email.com', '555-0008', '258 Spruce Ave', 'San Diego', 'CA', '92101', 'USA', '1993-06-07'),
('James', 'Taylor', 'james.t@email.com', '555-0009', '369 Willow Rd', 'Dallas', 'TX', '75201', 'USA', '1986-12-21'),
('Jennifer', 'Thomas', 'jennifer.t@email.com', '555-0010', '741 Ash Dr', 'San Jose', 'CA', '95101', 'USA', '1994-04-03');
GO

-- Insert Products
INSERT INTO Products (ProductName, SKU, CategoryID, Price, Cost, StockQuantity, Description)
VALUES 
('Laptop Pro 15"', 'LAP-PRO-15', 4, 1299.99, 899.99, 50, 'High-performance laptop with 16GB RAM'),
('Wireless Mouse', 'MSE-WRL-01', 4, 29.99, 15.99, 200, 'Ergonomic wireless mouse'),
('USB-C Cable 2M', 'CBL-USBC-2M', 1, 12.99, 5.99, 500, 'Fast charging USB-C cable'),
('Bluetooth Headphones', 'HDP-BLT-01', 2, 79.99, 45.99, 150, 'Noise-canceling headphones'),
('Phone Stand', 'ACC-PHS-01', 1, 19.99, 8.99, 300, 'Adjustable smartphone stand'),
('Mechanical Keyboard RGB', 'KBD-MCH-RGB', 4, 149.99, 89.99, 100, 'RGB mechanical gaming keyboard'),
('External SSD 1TB', 'SSD-EXT-1TB', 4, 129.99, 79.99, 75, 'Portable solid state drive'),
('Webcam HD 1080p', 'WBC-HD-1080', 1, 89.99, 55.99, 120, '1080p webcam with microphone'),
('Monitor 27" 4K', 'MON-27-4K', 4, 399.99, 279.99, 60, '4K UHD monitor'),
('LED Desk Lamp', 'LMP-DSK-LED', 3, 39.99, 22.99, 200, 'Adjustable LED desk lamp'),
('Wireless Charger', 'CHG-WRL-01', 1, 24.99, 12.99, 180, 'Fast wireless charging pad'),
('Graphics Tablet', 'TBL-GFX-01', 4, 199.99, 129.99, 45, 'Digital drawing tablet'),
('Portable Speaker', 'SPK-PRT-01', 2, 59.99, 34.99, 90, 'Bluetooth portable speaker'),
('Laptop Bag', 'BAG-LAP-01', 3, 44.99, 25.99, 150, 'Padded laptop backpack'),
('USB Hub 7-Port', 'HUB-USB-7P', 1, 34.99, 18.99, 110, '7-port powered USB hub');
GO

-- Insert Orders for 2022
SET IDENTITY_INSERT Orders OFF;
GO

INSERT INTO Orders (CustomerID, OrderDate, TotalAmount, OrderStatus, ShippingAddress, PaymentMethod, PaymentStatus)
VALUES 
(1, '2022-01-15 10:30:00', 1329.98, 'Completed', '123 Main St, New York, NY', 'Credit Card', 'Paid'),
(2, '2022-03-20 14:15:00', 109.98, 'Completed', '456 Oak Ave, Los Angeles, CA', 'PayPal', 'Paid'),
(3, '2022-05-10 09:45:00', 399.99, 'Completed', '789 Pine Rd, Chicago, IL', 'Credit Card', 'Paid'),
(4, '2022-08-22 16:20:00', 194.98, 'Completed', '321 Elm St, Houston, TX', 'Debit Card', 'Paid'),
(5, '2022-11-30 11:10:00', 89.99, 'Completed', '654 Maple Dr, Phoenix, AZ', 'Credit Card', 'Paid');
GO

-- Insert Orders for 2023
INSERT INTO Orders (CustomerID, OrderDate, TotalAmount, OrderStatus, ShippingAddress, PaymentMethod, PaymentStatus)
VALUES 
(1, '2023-01-10 13:30:00', 179.98, 'Completed', '123 Main St, New York, NY', 'Credit Card', 'Paid'),
(6, '2023-02-14 10:00:00', 1299.99, 'Completed', '987 Cedar Ln, Philadelphia, PA', 'Credit Card', 'Paid'),
(4, '2023-04-15 15:45:00', 449.97, 'Completed', '321 Elm St, Houston, TX', 'Debit Card', 'Paid'),
(7, '2023-06-20 12:30:00', 534.97, 'Completed', '147 Birch St, San Antonio, TX', 'PayPal', 'Paid'),
(5, '2023-07-22 09:15:00', 219.98, 'Completed', '654 Maple Dr, Phoenix, AZ', 'Credit Card', 'Paid'),
(2, '2023-10-05 14:50:00', 89.99, 'Completed', '456 Oak Ave, Los Angeles, CA', 'PayPal', 'Paid'),
(8, '2023-11-11 16:00:00', 274.97, 'Completed', '258 Spruce Ave, San Diego, CA', 'Credit Card', 'Paid'),
(3, '2023-12-20 10:30:00', 1329.98, 'Completed', '789 Pine Rd, Chicago, IL', 'Credit Card', 'Paid');
GO

-- Insert Orders for 2024
INSERT INTO Orders (CustomerID, OrderDate, TotalAmount, OrderStatus, ShippingAddress, PaymentMethod, PaymentStatus)
VALUES 
(1, '2024-01-05 11:20:00', 429.97, 'Completed', '123 Main St, New York, NY', 'Credit Card', 'Paid'),
(9, '2024-02-14 15:30:00', 1299.99, 'Completed', '369 Willow Rd, Dallas, TX', 'Credit Card', 'Paid'),
(2, '2024-03-25 10:15:00', 179.97, 'Completed', '456 Oak Ave, Los Angeles, CA', 'PayPal', 'Paid'),
(3, '2024-05-18 14:45:00', 1699.97, 'Completed', '789 Pine Rd, Chicago, IL', 'Credit Card', 'Paid'),
(10, '2024-06-30 09:00:00', 254.97, 'Completed', '741 Ash Dr, San Jose, CA', 'Debit Card', 'Paid'),
(4, '2024-08-09 13:30:00', 259.98, 'Completed', '321 Elm St, Houston, TX', 'Debit Card', 'Paid'),
(6, '2024-09-15 16:45:00', 594.96, 'Shipped', '987 Cedar Ln, Philadelphia, PA', 'Credit Card', 'Paid'),
(5, '2024-11-30 11:00:00', 129.99, 'Processing', '654 Maple Dr, Phoenix, AZ', 'Credit Card', 'Paid');
GO

-- Insert Orders for 2025
INSERT INTO Orders (CustomerID, OrderDate, TotalAmount, OrderStatus, ShippingAddress, PaymentMethod, PaymentStatus)
VALUES 
(1, '2025-01-03 10:15:00', 1329.98, 'Processing', '123 Main St, New York, NY', 'Credit Card', 'Paid'),
(3, '2025-01-04 14:30:00', 169.98, 'Pending', '789 Pine Rd, Chicago, IL', 'PayPal', 'Pending');
GO

-- Get OrderIDs to insert OrderItems (since IDENTITY values were generated)
-- We'll use a different approach - insert items directly referencing the orders

-- Insert OrderItems for all orders
-- We need to match OrderID with OrderDate from Orders table

-- For 2022 Orders
INSERT INTO OrderItems (OrderID, OrderDate, ProductID, Quantity, UnitPrice, Discount)
SELECT o.OrderID, o.OrderDate, 1, 1, 1299.99, 0
FROM Orders o WHERE o.CustomerID = 1 AND YEAR(o.OrderDate) = 2022;

INSERT INTO OrderItems (OrderID, OrderDate, ProductID, Quantity, UnitPrice, Discount)
SELECT o.OrderID, o.OrderDate, 2, 1, 29.99, 0
FROM Orders o WHERE o.CustomerID = 1 AND YEAR(o.OrderDate) = 2022;

INSERT INTO OrderItems (OrderID, OrderDate, ProductID, Quantity, UnitPrice, Discount)
SELECT o.OrderID, o.OrderDate, 4, 1, 79.99, 0
FROM Orders o WHERE o.CustomerID = 2 AND YEAR(o.OrderDate) = 2022;

INSERT INTO OrderItems (OrderID, OrderDate, ProductID, Quantity, UnitPrice, Discount)
SELECT o.OrderID, o.OrderDate, 2, 1, 29.99, 0
FROM Orders o WHERE o.CustomerID = 2 AND YEAR(o.OrderDate) = 2022;

INSERT INTO OrderItems (OrderID, OrderDate, ProductID, Quantity, UnitPrice, Discount)
SELECT o.OrderID, o.OrderDate, 9, 1, 399.99, 0
FROM Orders o WHERE o.CustomerID = 3 AND YEAR(o.OrderDate) = 2022;

INSERT INTO OrderItems (OrderID, OrderDate, ProductID, Quantity, UnitPrice, Discount)
SELECT o.OrderID, o.OrderDate, 6, 1, 149.99, 0
FROM Orders o WHERE o.CustomerID = 4 AND YEAR(o.OrderDate) = 2022;

INSERT INTO OrderItems (OrderID, OrderDate, ProductID, Quantity, UnitPrice, Discount)
SELECT o.OrderID, o.OrderDate, 5, 1, 19.99, 10
FROM Orders o WHERE o.CustomerID = 4 AND YEAR(o.OrderDate) = 2022;

INSERT INTO OrderItems (OrderID, OrderDate, ProductID, Quantity, UnitPrice, Discount)
SELECT o.OrderID, o.OrderDate, 8, 1, 89.99, 0
FROM Orders o WHERE o.CustomerID = 5 AND YEAR(o.OrderDate) = 2022;

-- For 2023 Orders (simplified - adding 2-3 items per order)
INSERT INTO OrderItems (OrderID, OrderDate, ProductID, Quantity, UnitPrice, Discount)
SELECT o.OrderID, o.OrderDate, 6, 1, 149.99, 0
FROM Orders o WHERE o.CustomerID = 1 AND YEAR(o.OrderDate) = 2023;

INSERT INTO OrderItems (OrderID, OrderDate, ProductID, Quantity, UnitPrice, Discount)
SELECT o.OrderID, o.OrderDate, 3, 1, 12.99, 0
FROM Orders o WHERE o.CustomerID = 1 AND YEAR(o.OrderDate) = 2023;

INSERT INTO OrderItems (OrderID, OrderDate, ProductID, Quantity, UnitPrice, Discount)
SELECT o.OrderID, o.OrderDate, 1, 1, 1299.99, 0
FROM Orders o WHERE o.CustomerID = 6 AND YEAR(o.OrderDate) = 2023;

-- Add more items for 2024 and 2025 orders
INSERT INTO OrderItems (OrderID, OrderDate, ProductID, Quantity, UnitPrice, Discount)
SELECT o.OrderID, o.OrderDate, 9, 1, 399.99, 0
FROM Orders o WHERE o.CustomerID = 1 AND o.OrderDate >= '2024-01-01' AND o.OrderDate < '2024-02-01';

INSERT INTO OrderItems (OrderID, OrderDate, ProductID, Quantity, UnitPrice, Discount)
SELECT o.OrderID, o.OrderDate, 2, 1, 29.99, 0
FROM Orders o WHERE o.CustomerID = 1 AND o.OrderDate >= '2024-01-01' AND o.OrderDate < '2024-02-01';

INSERT INTO OrderItems (OrderID, OrderDate, ProductID, Quantity, UnitPrice, Discount)
SELECT o.OrderID, o.OrderDate, 1, 1, 1299.99, 0
FROM Orders o WHERE o.CustomerID = 9 AND YEAR(o.OrderDate) = 2024;

INSERT INTO OrderItems (OrderID, OrderDate, ProductID, Quantity, UnitPrice, Discount)
SELECT o.OrderID, o.OrderDate, 1, 1, 1299.99, 0
FROM Orders o WHERE o.CustomerID = 1 AND YEAR(o.OrderDate) = 2025;

INSERT INTO OrderItems (OrderID, OrderDate, ProductID, Quantity, UnitPrice, Discount)
SELECT o.OrderID, o.OrderDate, 8, 1, 89.99, 0
FROM Orders o WHERE o.CustomerID = 3 AND YEAR(o.OrderDate) = 2025;

INSERT INTO OrderItems (OrderID, OrderDate, ProductID, Quantity, UnitPrice, Discount)
SELECT o.OrderID, o.OrderDate, 4, 1, 79.99, 0
FROM Orders o WHERE o.CustomerID = 3 AND YEAR(o.OrderDate) = 2025;
GO

-- Insert Customer Activity data
INSERT INTO CustomerActivity (CustomerID, ActivityDate, ActivityType, ProductID, SessionID)
VALUES 
(1, '2024-12-01 10:15:00', 'Login', NULL, 'SES001'),
(1, '2024-12-01 10:16:30', 'View', 1, 'SES001'),
(1, '2024-12-01 10:18:45', 'AddToCart', 1, 'SES001'),
(2, '2024-12-15 14:30:00', 'Login', NULL, 'SES002'),
(2, '2024-12-15 14:32:15', 'View', 9, 'SES002'),
(3, '2024-12-28 09:45:00', 'Login', NULL, 'SES003'),
(3, '2024-12-28 09:47:20', 'View', 4, 'SES003'),
(3, '2024-12-28 09:50:00', 'AddToCart', 4, 'SES003'),
(1, '2025-01-02 11:20:00', 'Login', NULL, 'SES004'),
(1, '2025-01-02 11:22:30', 'View', 1, 'SES004'),
(1, '2025-01-02 11:25:00', 'Purchase', 1, 'SES004'),
(4, '2025-01-03 15:10:00', 'Login', NULL, 'SES005'),
(4, '2025-01-03 15:12:45', 'View', 6, 'SES005');
GO

PRINT 'Sample data inserted successfully!';
GO
```

### Step 9: Query and Analyze Partitioned Data

```sql
-- ============================================
-- PARTITION ANALYSIS QUERIES
-- ============================================

-- 1. View partition distribution for Orders table
SELECT 
    OBJECT_NAME(p.object_id) AS TableName,
    p.partition_number AS PartitionNumber,
    p.rows AS RowCount,
    au.total_pages * 8 / 1024.0 AS TotalSpaceMB,
    au.used_pages * 8 / 1024.0 AS UsedSpaceMB,
    prv.value AS UpperBoundary
FROM sys.partitions p
INNER JOIN sys.allocation_units au ON p.partition_id = au.container_id
LEFT JOIN sys.partition_range_values prv ON p.partition_number = prv.boundary_id + 1
    AND prv.function_id = (SELECT function_id FROM sys.partition_functions WHERE name = 'PF_OrdersByYear')
WHERE OBJECT_NAME(p.object_id) = 'Orders'
    AND p.index_id IN (0,1)  -- Heap or clustered index only
ORDER BY p.partition_number;
GO

-- 2. Which partition does a specific date belong to?
SELECT 
    '2024-06-15' AS CheckDate,
    $PARTITION.PF_OrdersByYear('2024-06-15') AS PartitionNumber;
GO

SELECT 
    '2025-01-03' AS CheckDate,
    $PARTITION.PF_OrdersByYear('2025-01-03') AS PartitionNumber;
GO

-- 3. Orders summary by partition
SELECT 
    $PARTITION.PF_OrdersByYear(OrderDate) AS PartitionNumber,
    YEAR(OrderDate) AS OrderYear,
    COUNT(*) AS OrderCount,
    SUM(TotalAmount) AS TotalRevenue,
    AVG(TotalAmount) AS AvgOrderValue,
    MIN(OrderDate) AS FirstOrder,
    MAX(OrderDate) AS LastOrder
FROM Orders
GROUP BY $PARTITION.PF_OrdersByYear(OrderDate), YEAR(OrderDate)
ORDER BY PartitionNumber;
GO

-- 4. Query with partition elimination (scans only 2024 partition)
-- Enable actual execution plan to see partition elimination
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
GO

SELECT 
    o.OrderID,
    o.OrderDate,
    c.FirstName + ' ' + c.LastName AS CustomerName,
    o.TotalAmount,
    o.OrderStatus
FROM Orders o
INNER JOIN Customers c ON o.CustomerID = c.CustomerID
WHERE o.OrderDate >= '2024-01-01' AND o.OrderDate < '2025-01-01'
ORDER BY o.OrderDate DESC;
GO

SET STATISTICS IO OFF;
SET STATISTICS TIME OFF;
GO

-- 5. Top products by year (using partitioned tables)
SELECT 
    YEAR(oi.OrderDate) AS SalesYear,
    p.ProductName,
    SUM(oi.Quantity) AS TotalQuantitySold,
    SUM(oi.Subtotal) AS TotalRevenue,
    COUNT(DISTINCT oi.OrderID) AS NumberOfOrders
FROM OrderItems oi
INNER JOIN Products p ON oi.ProductID = p.ProductID
WHERE oi.OrderDate >= '2023-01-01'
GROUP BY YEAR(oi.OrderDate), p.ProductName
ORDER BY SalesYear DESC, TotalRevenue DESC;
GO

-- 6. Customer activity by month (monthly partitioned table)
SELECT 
    $PARTITION.PF_ActivityByMonth(ActivityDate) AS PartitionNumber,
    FORMAT(ActivityDate, 'yyyy-MM') AS ActivityMonth,
    COUNT(*) AS ActivityCount,
    COUNT(DISTINCT CustomerID) AS UniqueCustomers
FROM CustomerActivity
GROUP BY $PARTITION.PF_ActivityByMonth(ActivityDate), FORMAT(ActivityDate, 'yyyy-MM')
ORDER BY ActivityMonth DESC;
GO

-- 7. Compare partitioned vs non-partitioned query performance
-- Query entire table (all partitions)
SELECT COUNT(*) AS TotalOrders, SUM(TotalAmount) AS TotalRevenue
FROM Orders;
GO

-- Query specific partition
SELECT COUNT(*) AS TotalOrders, SUM(TotalAmount) AS TotalRevenue
FROM Orders
WHERE OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01';
GO
```

### Step 10: Advanced Partition Management

```sql
-- ============================================
-- PARTITION MAINTENANCE OPERATIONS
-- ============================================

-- 1. Add a new partition for 2027
-- First, split the partition function
ALTER PARTITION FUNCTION PF_OrdersByYear()
SPLIT RANGE ('2027-01-01');
GO

-- Verify new partition
SELECT 
    pf.name AS PartitionFunction,
    prv.boundary_id,
    prv.value AS BoundaryValue
FROM sys.partition_functions pf
LEFT JOIN sys.partition_range_values prv ON pf.function_id = prv.function_id
WHERE pf.name = 'PF_OrdersByYear'
ORDER BY prv.boundary_id;
GO

-- 2. Archive old data (create archive table for 2022)
-- Create archive table with same structure
CREATE TABLE Orders_Archive_2022
(
    OrderID INT,
    CustomerID INT NOT NULL,
    OrderDate DATETIME2 NOT NULL,
    TotalAmount DECIMAL(10,2) NOT NULL,
    OrderStatus NVARCHAR(20) NOT NULL,
    ShippingAddress NVARCHAR(200),
    BillingAddress NVARCHAR(200),
    PaymentMethod NVARCHAR(50),
    PaymentStatus NVARCHAR(20),
    ShippingMethod NVARCHAR(50),
    TrackingNumber NVARCHAR(100),
    Notes NVARCHAR(500),
    CreatedDate DATETIME2,
    LastModifiedDate DATETIME2,
    
    CONSTRAINT PK_Orders_Archive_2022 PRIMARY KEY (OrderID, OrderDate)
) ON [PRIMARY];
GO

-- Note: In Azure SQL Database, partition switching requires:
-- 1. Same filegroup (PRIMARY in our case)
-- 2. Same constraints and structure
-- 3. Table must be empty or partition must match

-- Switch partition (move 2022 data to archive)
-- First, verify which partition contains 2022 data
SELECT $PARTITION.PF_OrdersByYear('2022-06-15') AS Partition2022;
GO

-- Switch the partition
ALTER TABLE Orders 
SWITCH PARTITION 2 TO Orders_Archive_2022;
GO

-- Verify the switch
SELECT COUNT(*) AS ArchivedOrders FROM Orders_Archive_2022;
SELECT COUNT(*) AS RemainingOrders FROM Orders WHERE YEAR(OrderDate) = 2022;
GO

-- 3. Merge partitions (if needed to consolidate)
-- Example: Merge 2022 partition (after archiving)
-- This removes the 2022 boundary, combining partition 1 and 2
ALTER PARTITION FUNCTION PF_OrdersByYear()
MERGE RANGE ('2022-01-01');
GO

-- 4. Rebuild partitioned indexes
-- Rebuild specific partition
ALTER INDEX IX_Orders_CustomerID ON Orders
REBUILD PARTITION = 4;  -- Rebuild only 2024 partition
GO

-- Rebuild all partitions
ALTER INDEX IX_Orders_CustomerID ON Orders
REBUILD PARTITION = ALL;
GO
```

### Step 11: Performance Monitoring Queries

```sql
-- ============================================
-- PERFORMANCE MONITORING
-- ============================================

-- 1. Partition statistics
SELECT 
    OBJECT_SCHEMA_NAME(p.object_id) + '.' + OBJECT_NAME(p.object_id) AS TableName,
    i.name AS IndexName,
    p.partition_number,
    p.rows,
    au.total_pages,
    au.used_pages,
    au.data_pages,
    (au.total_pages * 8) / 1024.0 AS TotalSpaceMB,
    (au.used_pages * 8) / 1024.0 AS UsedSpaceMB,
    CASE 
        WHEN p.rows > 0 THEN (au.total_pages * 8 * 1024.0) / p.rows 
        ELSE 0 
    END AS BytesPerRow
FROM sys.partitions p
INNER JOIN sys.indexes i ON p.object_id = i.object_id AND p.index_id = i.index_id
INNER JOIN sys.allocation_units au ON p.partition_id = au.container_id
WHERE OBJECT_NAME(p.object_id) IN ('Orders', 'OrderItems')
    AND i.index_id <= 1  -- Clustered index or heap
ORDER BY TableName, p.partition_number;
GO

-- 2. Index fragmentation by partition
SELECT 
    OBJECT_NAME(ips.object_id) AS TableName,
    i.name AS IndexName,
    ips.partition_number,
    ips.avg_fragmentation_in_percent,
    ips.page_count,
    ips.avg_page_space_used_in_percent
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'DETAILED') ips
INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE OBJECT_NAME(ips.object_id) IN ('Orders', 'OrderItems')
    AND ips.index_id > 0
ORDER BY TableName, IndexName, partition_number;
GO

-- 3. Query execution statistics
SELECT 
    t.text AS QueryText,
    qs.execution_count,
    qs.total_elapsed_time / 1000000.0 AS TotalElapsedTimeSec,
    qs.total_worker_time / 1000000.0 AS TotalCPUTimeSec,
    qs.total_logical_reads,
    qs.total_logical_writes,
    qs.creation_time,
    qs.last_execution_time
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) t
WHERE t.text LIKE '%Orders%'
    AND t.text NOT LIKE '%sys.dm_exec_query_stats%'
ORDER BY qs.total_elapsed_time DESC;
GO
```

### Step 12: Business Intelligence Queries

```sql
-- ============================================
-- BUSINESS INTELLIGENCE QUERIES
-- ============================================

-- 1. Sales trend analysis by year
SELECT 
    YEAR(o.OrderDate) AS SalesYear,
    MONTH(o.OrderDate) AS SalesMonth,
    COUNT(DISTINCT o.OrderID) AS OrderCount,
    COUNT(DISTINCT o.CustomerID) AS UniqueCustomers,
    SUM(o.TotalAmount) AS TotalRevenue,
    AVG(o.TotalAmount) AS AvgOrderValue,
    SUM(oi.Quantity) AS TotalItemsSold
FROM Orders o
INNER JOIN OrderItems oi ON o.OrderID = oi.OrderID AND o.OrderDate = oi.OrderDate
GROUP BY YEAR(o.OrderDate), MONTH(o.OrderDate)
ORDER BY SalesYear, SalesMonth;
GO

-- 2. Customer lifetime value
SELECT 
    c.CustomerID,
    c.FirstName + ' ' + c.LastName AS CustomerName,
    c.Email,
    COUNT(DISTINCT o.OrderID) AS TotalOrders,
    SUM(o.TotalAmount) AS LifetimeValue,
    AVG(o.TotalAmount) AS AvgOrderValue,
    MIN(o.OrderDate) AS FirstPurchase,
    MAX(o.OrderDate) AS LastPurchase,
    DATEDIFF(DAY, MIN(o.OrderDate), MAX(o.OrderDate)) AS CustomerAgeDays
FROM Customers c
INNER JOIN Orders o ON c.CustomerID = o.CustomerID
GROUP BY c.CustomerID, c.FirstName, c.LastName, c.Email
ORDER BY LifetimeValue DESC;
GO

-- 3. Product performance analysis
SELECT 
    p.ProductID,
    p.ProductName,
    p.SKU,
    cat.CategoryName,
    p.Price,
    p.StockQuantity,
    COUNT(DISTINCT oi.OrderID) AS TimesOrdered,
    SUM(oi.Quantity) AS TotalQuantitySold,
    SUM(oi.Subtotal) AS TotalRevenue,
    AVG(oi.UnitPrice) AS AvgSellingPrice,
    MAX(oi.OrderDate) AS LastSoldDate
FROM Products p
LEFT JOIN Categories cat ON p.CategoryID = cat.CategoryID
LEFT JOIN OrderItems oi ON p.ProductID = oi.ProductID
GROUP BY p.ProductID, p.ProductName, p.SKU, cat.CategoryName, p.Price, p.StockQuantity
ORDER BY TotalRevenue DESC;
GO

-- 4. Monthly sales comparison (current vs previous year)
WITH MonthlySales AS (
    SELECT 
        YEAR(OrderDate) AS SalesYear,
        MONTH(OrderDate) AS SalesMonth,
        SUM(TotalAmount) AS MonthlyRevenue,
        COUNT(*) AS MonthlyOrders
    FROM Orders
    GROUP BY YEAR(OrderDate), MONTH(OrderDate)
)
SELECT 
    curr.SalesYear,
    curr.SalesMonth,
    curr.MonthlyRevenue AS CurrentYearRevenue,
    curr.MonthlyOrders AS CurrentYearOrders,
    prev.MonthlyRevenue AS PreviousYearRevenue,
    prev.MonthlyOrders AS PreviousYearOrders,
    curr.MonthlyRevenue - ISNULL(prev.MonthlyRevenue, 0) AS RevenueDifference,
    CASE 
        WHEN prev.MonthlyRevenue > 0 
        THEN ((curr.MonthlyRevenue - prev.MonthlyRevenue) / prev.MonthlyRevenue * 100)
        ELSE NULL 
    END AS GrowthPercentage
FROM MonthlySales curr
LEFT JOIN MonthlySales prev ON curr.SalesMonth = prev.SalesMonth 
    AND curr.SalesYear = prev.SalesYear + 1
ORDER BY curr.SalesYear, curr.SalesMonth;
GO

-- 5. Customer activity funnel analysis
SELECT 
    ActivityType,
    COUNT(*) AS ActivityCount,
    COUNT(DISTINCT CustomerID) AS UniqueCustomers,
    COUNT(DISTINCT CAST(ActivityDate AS DATE)) AS ActiveDays
FROM CustomerActivity
WHERE ActivityDate >= '2024-01-01'
GROUP BY ActivityType
ORDER BY ActivityCount DESC;
GO
```

### Step 13: Create Useful Views

```sql
-- ============================================
-- CREATE VIEWS FOR REPORTING
-- ============================================

-- 1. Order details view
CREATE OR ALTER VIEW vw_OrderDetails
AS
SELECT 
    o.OrderID,
    o.OrderDate,
    c.CustomerID,
    c.FirstName + ' ' + c.LastName AS CustomerName,
    c.Email AS CustomerEmail,
    o.TotalAmount,
    o.OrderStatus,
    o.PaymentMethod,
    o.PaymentStatus,
    o.ShippingAddress,
    COUNT(oi.OrderItemID) AS ItemCount,
    $PARTITION.PF_OrdersByYear(o.OrderDate) AS PartitionNumber
FROM Orders o
INNER JOIN Customers c ON o.CustomerID = c.CustomerID
LEFT JOIN OrderItems oi ON o.OrderID = oi.OrderID AND o.OrderDate = oi.OrderDate
GROUP BY o.OrderID, o.OrderDate, c.CustomerID, c.FirstName, c.LastName, c.Email,
         o.TotalAmount, o.OrderStatus, o.PaymentMethod, o.PaymentStatus, o.ShippingAddress;
GO

-- 2. Product sales summary view
CREATE OR ALTER VIEW vw_ProductSalesSummary
AS
SELECT 
    p.ProductID,
    p.ProductName,
    p.SKU,
    cat.CategoryName,
    p.Price AS CurrentPrice,
    p.StockQuantity,
    SUM(oi.Quantity) AS TotalSold,
    SUM(oi.Subtotal) AS TotalRevenue,
    COUNT(DISTINCT oi.OrderID) AS NumberOfOrders,
    AVG(oi.UnitPrice) AS AvgSellingPrice
FROM Products p
LEFT JOIN Categories cat ON p.CategoryID = cat.CategoryID
LEFT JOIN OrderItems oi ON p.ProductID = oi.ProductID
GROUP BY p.ProductID, p.ProductName, p.SKU, cat.CategoryName, p.Price, p.StockQuantity;
GO

-- Query the views
SELECT TOP 10 * FROM vw_OrderDetails ORDER BY OrderDate DESC;
SELECT TOP 10 * FROM vw_ProductSalesSummary ORDER BY TotalRevenue DESC;
GO
```

---

## Best Practices for Azure SQL Database Partitioning

### 1. Partition Key Selection
- Choose a column frequently used in WHERE clauses
- Use datetime columns for time-series data
- Ensure even data distribution across partitions

### 2. Partition Alignment
- Align related tables on the same partition key
- Use consistent partition functions across tables
- Maintain referential integrity with partitioned keys

### 3. Query Optimization
- Always filter on partition column
- Enable partition elimination
- Review execution plans regularly

### 4. Maintenance Strategy
- Archive old partitions regularly
- Add new partitions proactively
- Monitor partition sizes and growth

### 5. Index Strategy
- Create aligned indexes when possible
- Consider partition-level index maintenance
- Monitor index fragmentation per partition

### 6. Cost Optimization
- Use appropriate service tier for workload
- Consider scaling during peak times
- Archive historical data to reduce storage

---

## Troubleshooting Common Issues

### Issue 1: Cannot Create Filegroups
**Problem:** Error when trying to create custom filegroups
**Solution:** Azure SQL Database doesn't support custom filegroups. Use PRIMARY for all partitions.

```sql
-- CORRECT for Azure SQL Database
CREATE PARTITION SCHEME PS_MyScheme
AS PARTITION PF_MyFunction
ALL TO ([PRIMARY]);
```

### Issue 2: Partition Switch Fails
**Problem:** ALTER TABLE SWITCH fails with constraint errors
**Solution:** Ensure:
- Archive table has identical structure
- Same constraints (CHECK, NOT NULL)
- Same filegroup (PRIMARY)
- Compatible datatypes

### Issue 3: Poor Query Performance
**Problem:** Queries still scan all partitions
**Solution:** 
- Ensure WHERE clause includes partition column
- Check execution plan for partition elimination
- Verify partition function boundaries

```sql
-- BAD (scans all partitions)
SELECT * FROM Orders WHERE CustomerID = 1;

-- GOOD (partition elimination)
SELECT * FROM Orders 
WHERE CustomerID = 1 
  AND OrderDate >= '2024-01-01' 
  AND OrderDate < '2025-01-01';
```

### Issue 4: Connection Timeout
**Problem:** Connection times out from Azure Data Studio
**Solution:**
- Verify firewall rules include your IP
- Check if server allows Azure services
- Ensure correct server name with `.database.windows.net`

### Issue 5: DTU/vCore Limits
**Problem:** Queries are slow or throttled
**Solution:**
- Scale up service tier
- Optimize queries with partition elimination
- Consider upgrading to higher performance tier

---

## Monitoring and Optimization

### Query Performance Monitoring

```sql
-- Monitor query duration
SELECT 
    CAST(query_stats.query_hash AS VARCHAR(20)) AS QueryHash,
    COUNT(*) AS ExecutionCount,
    AVG(query_stats.total_elapsed_time / 1000.0) AS AvgDurationMS,
    MAX(query_stats.total_elapsed_time / 1000.0) AS MaxDurationMS,
    SUM(query_stats.total_logical_reads) AS TotalLogicalReads
FROM sys.dm_exec_query_stats AS query_stats
CROSS APPLY sys.dm_exec_sql_text(query_stats.sql_handle) AS query_text
WHERE query_text.text LIKE '%Orders%'
    AND query_text.text NOT LIKE '%sys.dm_exec%'
GROUP BY query_stats.query_hash
ORDER BY AvgDurationMS DESC;
GO
```

### Storage Monitoring

```sql
-- Check database size
SELECT 
    database_name,
    SUM(reserved_page_count) * 8.0 / 1024 AS DatabaseSizeMB
FROM sys.dm_db_partition_stats
GROUP BY database_name;
GO
```

---

## Additional Resources

### Official Documentation
- **Azure SQL Database:** https://docs.microsoft.com/azure/azure-sql/database/
- **Partitioning:** https://docs.microsoft.com/sql/relational-databases/partitions/partitioned-tables-and-indexes
- **Azure Data Studio:** https://docs.microsoft.com/sql/azure-data-studio/

### Useful Links
- Azure SQL Database pricing: https://azure.microsoft.com/pricing/details/sql-database/
- Performance best practices: https://docs.microsoft.com/azure/azure-sql/database/performance-guidance
- Monitoring: https://docs.microsoft.com/azure/azure-sql/database/monitor-tune-overview

---

## Summary Checklist

✅ **Azure Infrastructure Setup:**
- Created Azure SQL Database server
- Created EcommerceDB database
- Configured firewall rules
- Created Windows Server VM in Azure
- Connected to Windows Server via RDP

✅ **Azure Data Studio Setup:**
- Installed Azure Data Studio on Windows Server
- Configured Windows Server for optimal performance
- Connected to Azure SQL Database from Windows Server

✅ **Partitioning Implemented:**
- Created partition functions (yearly & monthly)
- Created partition schemes
- Created partitioned tables (Orders, OrderItems, CustomerActivity)
- Inserted sample data across multiple years

✅ **Advanced Features:**
- Created aligned indexes
- Implemented partition maintenance
- Built business intelligence queries
- Created reporting views

✅ **Best Practices Applied:**
- Partition elimination in queries
- Proper partition key selection
- Index alignment strategy
- Regular monitoring queries

**You now have a fully functional Azure SQL Database with partitioned tables, accessible from your Windows Server VM with Azure Data Studio!**
