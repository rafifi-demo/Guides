# SQL Server on AWS → Azure Arc Pay-As-You-Go
## A Technical Guide for Transitioning License Billing Without Migrating Your Workload

---

## Overview

### What We Are Trying to Achieve

If you are running SQL Server on AWS today using a **license-included AMI**, AWS bundles the SQL Server license cost into your EC2 hourly rate. You pay for compute and license together — and you have limited visibility or control over how that license is tracked and governed.

This guide shows you how to **move SQL Server license billing from AWS into Azure** using **Azure Arc** — without immediately migrating your database or disrupting your running workload.

The outcome: a second EC2 instance running SQL Server 2022 on a plain Windows Server AMI, with the license billed **hourly through your Azure subscription** via Azure Arc PAYG. Your data is continuously synced from source to target using log shipping. When you are ready, you cut over, redirect your application, and terminate the old instance — stopping AWS license billing at that moment.

### How We Are Going to Do It

The approach has seven stages:

| Stage | What Happens |
|---|---|
| **Prepare the source database** | Ensure the database is in FULL recovery model (required for log shipping) |
| **Prepare Azure** | Register the resource providers and create the Arc infrastructure |
| **Build the target** | Launch a second EC2, install SQL Server 2022 with PAYG billing via the installer wizard |
| **Validate Arc** | Confirm the machine is registered in Azure Arc and the license type is Pay-as-you-go |
| **Configure log shipping** | Begin continuous transaction log sync from source to target |
| **Cut over** | Apply the final log backup, bring the target online, redirect connections |
| **Test** | Verify data integrity, write access, and billing confirmation in Azure Portal |

### Customer Impact

For customers running SQL Server on AWS, this approach delivers three immediate benefits:

1. **License portability** — SQL Server licensing moves into Azure where it can be governed, tracked, and audited centrally alongside other Azure resources.
2. **No forced migration** — You do not need to lift and shift your workload to Azure. The database stays in AWS. Only the license billing changes.
3. **Near-zero downtime cutover** — Log shipping keeps source and target in near-continuous sync. Cutover is a matter of applying a final log backup and redirecting a connection string.

---

## Part 1: Prepare the Source Database

### Objective

Log shipping — the data sync method used in this guide — requires the source database to be in **FULL recovery model**. SIMPLE recovery model truncates the transaction log automatically, which breaks log shipping. This is the only prerequisite change needed on the source. Everything else — the data, the schema, the running application — stays exactly as it is.

### How

Connect to your source SQL Server instance via SSMS, open a New Query, and run:

```sql
-- Set FULL recovery model (required for log shipping)
ALTER DATABASE <your_database_name> SET RECOVERY FULL;
GO
```

Verify the change was applied:

```sql
SELECT name, recovery_model_desc
FROM sys.databases
WHERE name = '<your_database_name>';
```

**Expected output:** `recovery_model_desc` returns `FULL`.

> This statement is safe to run even if the database is already in FULL recovery model — it is idempotent.

---

## Part 2: Prepare the Azure Environment

### Objective

Before the target instance can register with Azure Arc and activate PAYG billing, three things need to be in place in Azure: a **resource group** to hold the Arc resources, the required **resource providers** registered in your subscription, and a **Log Analytics workspace** for telemetry. Without these, the Arc extension installed during SQL Server setup has nowhere to register and the billing link cannot be established.

### How

Run the following from Azure CLI (Cloud Shell or a local terminal with `az` installed and authenticated).

#### 2.1 Create a Resource Group

This is the container for all Arc-related resources created in this guide.

```bash
az group create \
  --name <your_resource_group> \
  --location <your_azure_region>
```

#### 2.2 Register Required Resource Providers

These four providers enable Azure to manage hybrid machines, Arc SQL Server instances, connectivity, and policy. Registration is a one-time operation per subscription.

```bash
az provider register --namespace Microsoft.HybridCompute
az provider register --namespace Microsoft.HybridConnectivity
az provider register --namespace Microsoft.AzureArcData
az provider register --namespace Microsoft.GuestConfiguration
```

Verify all four are registered before moving on:

```bash
az provider show --namespace Microsoft.HybridCompute --query registrationState -o tsv
az provider show --namespace Microsoft.HybridConnectivity --query registrationState -o tsv
az provider show --namespace Microsoft.AzureArcData --query registrationState -o tsv
az provider show --namespace Microsoft.GuestConfiguration --query registrationState -o tsv
```

Each should return `Registered`. Registration can take 1–2 minutes — re-run if any show `Registering`.

#### 2.3 Create a Log Analytics Workspace

This workspace receives telemetry from the Arc-connected machine, including SQL Server health and activity data.

```bash
az monitor log-analytics workspace create \
  --resource-group <your_resource_group> \
  --workspace-name <your_log_analytics_workspace> \
  --location <your_azure_region>
```

---

## Part 3: Build the Target Instance with SQL Server PAYG

### Objective

This is the core of the entire guide. The goal is to launch a **plain Windows Server EC2** — no SQL Server license baked into the AMI — and install SQL Server 2022 with PAYG billing linked to Azure Arc. When you select the PAYG option in the SQL Server installer, it handles everything in a single step: SQL Server is installed, the Azure Extension for SQL Server is deployed, and the Azure Connected Machine Agent is started automatically. The machine self-registers with Arc — no separate onboarding required. From that moment, the SQL Server license is billed hourly through your Azure subscription.

### How

#### 3.1 Launch the Target EC2 Instance

**Goal:** Provision a clean Windows Server instance — no SQL Server included — that will become the Arc-managed target.

In the AWS Console, launch a new EC2 instance with the following configuration:

| Setting | Value |
|---|---|
| Name | `sql-target` |
| AMI | Windows Server 2022 Base (no SQL Server pre-installed) |
| Instance type | `m5.xlarge` or equivalent (4 vCPU, 16 GB RAM minimum) |
| Key pair | Same key pair used for the source instance |
| Storage | 100 GB gp3 |

Apply these inbound security group rules:

| Type | Protocol | Port | Source |
|---|---|---|---|
| RDP | TCP | 3389 | 0.0.0.0/0 |
| MS SQL | TCP | 1433 | 0.0.0.0/0 |

Once the instance is running, connect via RDP using the Administrator credentials retrieved from the EC2 console. Ensure the security group rules allow traffic on the required ports between source and target — particularly port 445 (SMB) for log shipping and port 1433 (SQL) for database access.

#### 3.2 Create a Service Principal

**Goal:** Generate Azure credentials that the SQL Server installer can use to authenticate and register the machine with Arc — without requiring interactive sign-in.

First, retrieve your subscription ID:

```bash
az account show --query id -o tsv
```

Then create a Service Principal scoped to your resource group:

```bash
az ad sp create-for-rbac \
  --name "<your_service_principal_name>" \
  --role "Contributor" \
  --scopes /subscriptions/<your_subscription_id>/resourceGroups/<your_resource_group>
```

The output looks like this:

```json
{
  "appId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "displayName": "<your_service_principal_name>",
  "password": "xxxx~xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "tenant": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> **Save these values immediately.** The password is only shown once and cannot be retrieved.

| Parameter | Source |
|---|---|
| Azure Service Principal ID | `appId` |
| Azure Service Principal Secret | `password` |
| Azure Subscription ID | output of `az account show` |
| Azure Tenant ID | `tenant` |
| Azure Resource Group | `<your_resource_group>` |
| Azure Region | `<your_azure_region>` |

#### 3.3 Download and Install SQL Server 2022 with PAYG

**Goal:** Install SQL Server 2022 on the target and activate Pay-as-you-go billing through Azure Arc — all in a single installation.

Download the SQL Server 2022 installer from Microsoft's site, run it, and work through the setup wizard:

```
https://info.microsoft.com/ww-landing-sql-server-2022.html
```

Key selections in the wizard:

1. **Edition** — Select **Use pay-as-you-go billing through Microsoft Azure | Standard**
2. **Azure Extension credentials** — Enter the Service Principal values from section 3.2:

| Field | Value |
|---|---|
| Azure Service Principal ID | your `appId` |
| Azure Service Principal Secret | your `password` |
| Azure Subscription ID | your subscription ID |
| Azure Resource Group | `<your_resource_group>` |
| Azure Region | `<your_azure_region>` |
| Azure Tenant ID | your `tenant` |

3. **Features** — Select **Database Engine Services**, leave everything else at defaults
4. **Authentication** — Set **Mixed Mode**, assign a strong SA password, and add the current Windows user as an administrator
5. **Install** — Review and confirm; all features should complete with **Succeeded**

> **Why no separate Arc agent install is needed:** Selecting PAYG billing in the wizard installs the Azure Extension for SQL Server, which in turn installs and starts the Azure Connected Machine Agent automatically. Arc onboarding is embedded in the SQL Server installer — there is nothing additional to configure.

#### 3.4 Install SSMS and Verify SQL Server

**Goal:** Confirm SQL Server is running and accessible on the target before moving data.

Download and install SQL Server Management Studio (SSMS):

```
https://aka.ms/ssmsfullsetup
```

Once installed, open SSMS, connect to `localhost` using Windows Authentication, and confirm the instance appears in Object Explorer.

---

## Part 4: Validate Arc Registration and PAYG Billing

### Objective

Before moving any data, confirm two things in the Azure Portal: the target machine is registered and **Connected** in Azure Arc, and the SQL Server license type is explicitly set to **Pay-as-you-go**. If either is not in place, data migration and cutover will succeed technically — but the billing change will not have taken effect.

### How

#### 4.1 Verify Arc Registration

**Goal:** Confirm the target machine is visible and connected in Azure Arc.

In the Azure Portal, navigate to **Azure Arc → Infrastructure → Machines** and confirm the target EC2 instance appears with status **Connected**. You can also check under **Data services → SQL Server instances**.

Alternatively, verify from the CLI:

```bash
az connectedmachine list --resource-group <your_resource_group> -o table
```

#### 4.2 Verify the License Type is Pay-as-you-go

**Goal:** Confirm the SQL Server license is billed through Azure, not AWS.

In the Azure Portal, go to **Azure Arc → Data services → SQL Server instances**, select your instance, and navigate to **Settings → Properties**. Confirm **License Type** shows **Pay-as-you-go**. This was set automatically by the installer.

#### 4.3 Confirm Billing is Active

**Goal:** Verify that hourly charges are appearing in Azure Cost Management for the Arc-registered SQL Server.

In the Azure Portal, open your resource group and go to **Cost Management → Cost analysis → Cost by resource**. Look for resource type `Microsoft.AzureArcData/SqlServerInstances`.

> Hourly charges appear within 24 hours of registration. If you just onboarded, check back the following day — this is expected.

---

## Part 5: Sync Data Using Log Shipping

### Objective

Log shipping bridges the source and target during the transition period. It continuously backs up the transaction log on the source and restores those backups on the target, keeping both databases in near-continuous sync. Using **STANDBY** mode on the target means you can query it read-only between restores — which lets you verify data is arriving correctly before committing to cutover. When you are ready to cut over, you apply one final log backup and the target becomes the authoritative database.

### How

#### 5.1 Create the Shared Backup Folder

**Goal:** Set up a shared network location on the source that the target can access to pick up log backup files.

On the **source instance**, create and share the backup folder:

```powershell
New-Item -ItemType Directory -Path "C:\SQLBackups"
New-SmbShare -Name "SQLBackups" -Path "C:\SQLBackups" -FullAccess "Everyone"
```

On the **target instance**, map a network drive to the source share using the source's Private IP (found in the EC2 console):

```powershell
$cred = Get-Credential   # enter source Administrator username and password when prompted
New-PSDrive -Name "Z" -PSProvider FileSystem -Root "\\<source_private_ip>\SQLBackups" -Persist -Credential $cred
```

#### 5.2 Take a Full Backup on the Source

**Goal:** Create the baseline backup that seeds the target database. Log shipping can only begin from a full backup restore.

On the source in SSMS:

```sql
BACKUP DATABASE <your_database_name>
TO DISK = 'C:\SQLBackups\<your_database_name>_Full.bak'
WITH FORMAT, COMPRESSION, STATS = 10;
```

#### 5.3 Restore the Full Backup on the Target

**Goal:** Seed the target with the source's data and put the database into STANDBY mode so it can receive log backups while remaining queryable.

On the target, first copy the backup file locally to avoid restoring directly over the network:

```powershell
New-Item -Path "C:\Temp" -ItemType Directory -Force
Copy-Item "\\<source_private_ip>\SQLBackups\<your_database_name>_Full.bak" "C:\Temp\<your_database_name>_Full.bak"
```

Then restore with STANDBY in SSMS on the target:

```sql
RESTORE DATABASE <your_database_name>
FROM DISK = 'C:\Temp\<your_database_name>_Full.bak'
WITH STANDBY = 'C:\Temp\<your_database_name>_Undo.bak',
     MOVE '<your_database_name>' TO 'C:\Program Files\Microsoft SQL Server\MSSQL16.MSSQLSERVER\MSSQL\DATA\<your_database_name>.mdf',
     MOVE '<your_database_name>_log' TO 'C:\Program Files\Microsoft SQL Server\MSSQL16.MSSQLSERVER\MSSQL\DATA\<your_database_name>_log.ldf',
     REPLACE,
     STATS = 10;
```

> **STANDBY vs NORECOVERY:** STANDBY allows read-only queries on the target between log restores — useful for validating data before cutover. NORECOVERY keeps the database in a pure restoring state with no query access. Use STANDBY.

#### 5.4 Configure Log Shipping on the Source

**Goal:** Enable the source to automatically produce transaction log backups on a regular schedule, so the target stays in continuous sync.

In SSMS on the source, right-click your database → **Properties → Transaction Log Shipping**. Enable it as a primary database and configure the backup settings:

| Setting | Value |
|---|---|
| Backup folder | `C:\SQLBackups` |
| Network path | `\\<source_private_ip>\SQLBackups` |
| Schedule | Every 1 minute (adjust for production) |

> For a basic single-instance setup, you do not need to configure the secondary server connection in the wizard. The source will produce log backup files continuously; the target will consume them manually or via scheduled job.

#### 5.5 Verify the Sync is Working

**Goal:** Confirm that data written on the source is arriving on the target within the log shipping interval.

Write a test row on the source:

```sql
USE <your_database_name>;
INSERT INTO <your_table_name> (<columns>) VALUES (<test_values>);
```

Wait 2–3 minutes, then on the target, check the restore history to confirm log restores are occurring:

```sql
SELECT TOP 10
    destination_database_name,
    restore_date,
    restore_type
FROM msdb.dbo.restorehistory
WHERE destination_database_name = '<your_database_name>'
ORDER BY restore_date DESC;
```

Then query the target database directly to confirm the test row arrived:

```sql
USE <your_database_name>;
SELECT * FROM <your_table_name>;
```

If the row is visible, the two instances are in near-sync and you are ready for cutover.

---

## Part 6: Cutover

### Objective

Cutover transitions the target from a read-only replica into the authoritative production database. The process is ordered to eliminate data loss: stop writes on the source first, take one final log backup, apply it to the target with `WITH RECOVERY` to bring it fully online, then redirect application connections. Terminating the source instance at the end stops AWS license billing immediately.

### How

#### 6.1 Stop Writes and Take a Final Log Backup

**Goal:** Freeze the source database and capture every last transaction so nothing is lost in the transition.

On the source in SSMS:

```sql
-- Prevent new connections
ALTER DATABASE <your_database_name> SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
GO

-- Take the final transaction log backup
BACKUP LOG <your_database_name>
TO DISK = 'C:\SQLBackups\<your_database_name>_Final.bak'
WITH STATS = 10;
```

#### 6.2 Apply the Final Log Backup and Bring the Target Online

**Goal:** Synchronise the target to the last committed transaction and bring it out of STANDBY into a fully read-write state.

Copy the final backup to the target, then apply it with `WITH RECOVERY`:

```sql
RESTORE LOG <your_database_name>
FROM DISK = 'C:\Temp\<your_database_name>_Final.bak'
WITH RECOVERY;
```

If the database is still in STANDBY after this step:

```sql
USE master;
GO
RESTORE DATABASE <your_database_name> WITH RECOVERY;
```

> `WITH RECOVERY` permanently ends log shipping. After this point, no further log backups can be applied. Only run this when you are committed to completing the cutover.

#### 6.3 Redirect Application Connections

**Goal:** Point your application at the target so it becomes the live database.

Update the connection string in your application from the source private IP to the target private IP:

```
Server=<target_ec2_private_ip>;Database=<your_database_name>;...
```

#### 6.4 Terminate the Source Instance

**Goal:** Stop AWS billing for the license-included SQL Server AMI.

Stop and terminate the source EC2 instance from the AWS console. AWS billing for the SQL Server license stops immediately. The license is now billed exclusively through Azure Arc.

---

## Part 7: Validate the Completed Migration

### Objective

After cutover, run three targeted checks to confirm the migration is complete: the target is reachable on port 1433, data arrived intact, and writes are working. Also confirm in Azure Portal that the Arc billing is active. Do not declare success until all three pass.

### How

#### 7.1 Connectivity Test

**Goal:** Confirm SQL Server on the target is reachable on port 1433.

```powershell
Test-NetConnection -ComputerName <target_ec2_private_ip> -Port 1433
```

**Expected:** `TcpTestSucceeded : True`

#### 7.2 Data Integrity Test

**Goal:** Confirm all data arrived on the target, including any rows written on the source during the sync period.

In SSMS on the target:

```sql
USE <your_database_name>;

-- Confirm total row count matches expectation
SELECT COUNT(*) AS TotalRows FROM <your_table_name>;

-- Check most recent records
SELECT TOP 5 * FROM <your_table_name> ORDER BY <date_column> DESC;
```

#### 7.3 Write Test

**Goal:** Confirm the database is fully online and accepting writes (not still in STANDBY).

```sql
USE <your_database_name>;

INSERT INTO <your_table_name> (<columns>)
VALUES (<post_migration_test_values>);

SELECT * FROM <your_table_name> WHERE <condition>;
```

**Expected:** 1 row returned.

> If you receive "database is read-only", the database is still in STANDBY. Run `RESTORE DATABASE <your_database_name> WITH RECOVERY;` and retry.

#### 7.4 Azure Arc PAYG Billing Confirmation

**Goal:** Confirm Azure is billing the SQL Server license and the license type is correctly set.

In the Azure Portal, go to **Azure Arc → SQL Server instances**, select your instance, and confirm **License type: Pay-as-you-go**. Then check **Cost Management → Cost analysis → Cost by resource** and filter by `Microsoft.AzureArcData/SqlServerInstances`.

Hourly charges appear within 24 hours of onboarding.

---

## Summary

| Step | What Happened |
|---|---|
| Source prepared | Source database set to FULL recovery model — log shipping prerequisite confirmed |
| Azure prepared | Resource group and providers registered, Log Analytics workspace created |
| Target built | Plain Windows Server AMI + SQL Server 2022 installed with PAYG + Arc agent — all in one step |
| Arc registered | Machine auto-registered in Azure Arc during SQL Server setup — no separate agent install required |
| PAYG enabled | SQL Server license set to Pay-as-you-go during installation wizard — no additional configuration needed |
| Validated | Arc registration, PAYG license type, and billing confirmed in Azure Portal |
| Log shipping configured | Shared folder created, full backup taken, STANDBY restore applied, continuous log sync running |
| Live sync verified | New data written on source confirmed visible on target via STANDBY read access |
| Cutover completed | Final log applied with RECOVERY, database brought fully online, connections redirected, source terminated |
| Post-cutover tested | Data intact, writes working, Arc billing confirmed |

---

## Key Design Decisions Explained

**Why keep SQL Server on AWS instead of migrating to Azure?**
This approach is intentional for customers who are not ready to move their compute workload to Azure — whether due to latency requirements, existing dependencies, contract commitments, or a phased migration plan. Azure Arc PAYG does not require the workload to move. It only requires the machine to be registered with Arc, which the SQL Server installer handles automatically.

**Why log shipping instead of backup/restore or Azure Database Migration Service?**
Log shipping works on all SQL Server editions, requires no additional software or Azure services, and is fully native to SQL Server. It supports near-continuous sync, making cutover a matter of minutes rather than hours. STANDBY mode adds the ability to query the target while sync is running — a practical advantage when validating data before committing to cutover.

**Why a Service Principal instead of interactive sign-in?**
The SQL Server setup wizard runs non-interactively. A Service Principal scoped to the resource group is the correct credential type — it avoids storing personal credentials in installer logs and can be revoked after setup completes.

**When does AWS billing stop?**
The moment you terminate the source EC2 instance. The license-included AMI charges are tied to the instance's running state. Once terminated, AWS stops billing for the SQL Server license immediately.
