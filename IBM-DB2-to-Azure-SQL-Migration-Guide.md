# IBM DB2 to Microsoft Azure SQL Migration Guide

A generic, reusable technical guide for migrating IBM DB2 workloads to Microsoft Azure SQL Managed Instance (SQL MI) or Azure SQL Database.

---

## Overview

This guide walks through the end-to-end process of migrating from IBM DB2 to Azure SQL. It assumes the DB2 source environment is already up and running with an existing schema and data. The guide covers assessment, schema conversion, data migration, and validation — with tooling options at each stage so teams can choose the approach that fits their requirements.

**Target audiences:** Solution architects, database engineers, and migration leads.

**Supported sources:** IBM DB2 LUW (Linux, Unix, Windows) versions 9.7 and above.

**Supported targets:**
- Azure SQL Managed Instance (recommended for lift-and-shift with maximum SQL Server compatibility)
- Azure SQL Database (recommended for cloud-native, serverless, or elastic workloads)

---

## Prerequisites

Before starting, confirm the following are in place:

| Item | Details |
|---|---|
| Azure subscription | Contributor or Owner role on the target resource group |
| Azure CLI | Installed and logged in (`az login`) |
| DB2 source access | Hostname/IP, port (default 50000), database name, and credentials with at minimum CONNECT authority |
| Windows machine | Required for SSMA — a Windows Server or desktop VM with network access to both DB2 and Azure SQL |
| SQL Server Management Studio (SSMS) | Installed on the Windows machine |
| SQL Server Migration Assistant for DB2 (SSMA) | Installed on the Windows machine |
| Azure SQL target | SQL MI instance or Azure SQL Database provisioned and in Ready state |
| Target database | Empty database pre-created on the Azure SQL target matching your source database name |

> **Network connectivity:** SSMA must be able to reach the DB2 host on port 50000 and the Azure SQL target on port 1433. If DB2 is on-premises, ensure VPN or ExpressRoute is in place, or run SSMA from a VM inside the same network as DB2.

---

## Phase 1 — Assessment

**Goal:** Understand the scope and complexity of the migration before touching anything in production.

### Step 1.1 — Inventory the DB2 source

**What and why:** Before running any migration tooling, capture a snapshot of what exists in the DB2 source — object counts by type, estimated row counts, and total data volume. This baseline serves two purposes later in this guide: the object counts are used in Step 2.4 to verify the schema was fully deployed to Azure SQL, and the data volume helps right-size the Azure SQL target and estimate migration duration. Row counts are captured again more precisely in Step 3.1 immediately before data migration.

Connect to your DB2 instance and record the output of each query:

```sql
-- Object counts by type (tables, views, procedures, triggers, indexes)
SELECT TYPE, COUNT(*) AS ObjectCount
FROM SYSCAT.OBJECTS
WHERE SCHEMA = 'YOUR_SCHEMA'
GROUP BY TYPE
ORDER BY TYPE;

-- Table list with estimated row counts
SELECT TABNAME, CARD AS EstimatedRows
FROM SYSCAT.TABLES
WHERE TABSCHEMA = 'YOUR_SCHEMA'
ORDER BY TABNAME;

-- Stored procedures
SELECT PROCNAME, LANGUAGE
FROM SYSCAT.PROCEDURES
WHERE PROCSCHEMA = 'YOUR_SCHEMA';

-- Triggers
SELECT TRIGNAME, TRIGEVENT, TRIGTIME
FROM SYSCAT.TRIGGERS
WHERE TRIGSCHEMA = 'YOUR_SCHEMA';

-- Total data size (GB)
SELECT SUM(FPAGES) * 4 / 1024.0 / 1024.0 AS DataSizeGB
FROM SYSCAT.TABLESPACES;
```

Save the output — you will reference the object counts in Step 2.4 and the data size when selecting the Azure SQL tier and migration tooling.

### Step 1.2 — Run SSMA Schema Conversion Assessment

**What and why:** SSMA reads every object in your DB2 schema and attempts to convert it to Azure SQL-compatible T-SQL. Nothing is written to either database at this stage — this is purely an analysis pass. The output is a compatibility report that tells you exactly which objects will convert automatically, which need review, and which require manual rewriting. This report becomes the foundation for estimating migration effort and is the primary deliverable of the assessment phase.

**How to run it:**

1. Open SSMA → **File → New Project** — set the target to **Azure SQL Managed Instance** or **Azure SQL Database**
2. Connect to DB2: **File → Connect to DB2** — enter the hostname, port (50000), database name, and credentials
3. In the schema explorer, expand the server, locate your schema, and check the checkbox next to it
4. Right-click the schema → **Convert Schema** — SSMA processes every object and marks each with a status:
   - **No overlay** — clean conversion, ready to deploy
   - **Yellow triangle** — converted with a warning, review before deploying to production
   - **Red X** — could not be converted automatically; a manual T-SQL rewrite is needed before deploying
5. Click **Create Report** in the toolbar — save the HTML report that opens in your browser

> **Using the report:** The report lists every flagged object with a specific explanation. Share it with stakeholders before committing to a timeline — the count of Red-flagged objects directly indicates developer effort needed before migration can proceed.

### Step 1.3 — Key compatibility areas to review

| DB2 Feature | Azure SQL Equivalent | SSMA Handles? |
|---|---|---|
| `FETCH FIRST N ROWS ONLY` | `SELECT TOP N` or `FETCH NEXT` | Yes |
| `CURRENT TIMESTAMP` | `GETDATE()` / `SYSDATETIME()` | Yes |
| `POSSTR(str, sub)` | `CHARINDEX(sub, str)` | Yes |
| `GRAPHIC` / `VARGRAPHIC` | `NCHAR` / `NVARCHAR` | Yes |
| `VALUES INTO` in procedures | `SELECT ... INTO` | Partial — review Yellow flags |
| `GENERATED ALWAYS AS IDENTITY` | `IDENTITY(1,1)` | Yes |
| `SEQUENCE` objects | `SEQUENCE` (SQL MI) | Yes for SQL MI, limited for SQL DB |
| Row-level security via labels | Row-Level Security policies | Manual rewrite required |
| DB2 XML data type | SQL Server XML type | Partial |
| Linked server references | Elastic Query or manual refactor | Manual |

---

## Phase 2 — Schema Conversion

**Goal:** Convert the DB2 DDL (tables, views, stored procedures, triggers, indexes) to Azure SQL-compatible T-SQL and deploy it to the target database.

### Tooling Options

| Tool | Best For | Notes |
|---|---|---|
| **SSMA for DB2** | Most migrations — automated conversion with inline review | Free, Windows-only, handles schema and data; does not support CDC |
| **Manual T-SQL rewrite** | Small schemas or objects with complex DB2-specific logic | Full control, time-intensive |
| **Striim** | Enterprises requiring schema + live data sync in one pipeline | Commercial, supports CDC, requires separate licensing |

**Recommendation:** Use SSMA for schema conversion in most cases. It is free, purpose-built for this migration path, and produces an auditable conversion report. Reserve manual rewrites for objects SSMA flags as Red (unresolvable errors).

### Step 2.1 — Pre-conversion checklist

Before deploying the converted schema to the target:

- [ ] Target database is created and empty
- [ ] SSMA is connected to both DB2 source and Azure SQL target
- [ ] Schema checkbox is checked in the DB2 Metadata Explorer
- [ ] Convert Schema has been run and the assessment report has been reviewed
- [ ] All Red-flagged objects have been manually corrected in SSMA's T-SQL editor

### Step 2.2 — Deploy the converted schema via SSMA

**What and why:** SSMA's "Synchronize with Database" takes the converted T-SQL it built during schema conversion and creates all the objects (tables, views, stored procedures, indexes) inside the empty target database on Azure SQL. Think of it as SSMA generating and executing the CREATE statements on your behalf. This step does not move any data — it only creates the structure.

**How to run it:**

1. In SSMA, locate your schema under the Azure SQL target in the right-hand explorer → right-click → **Synchronize with Database**
2. Review the confirmation dialog — it lists every object about to be created — then click **OK**
3. SSMA executes the CREATE statements and logs progress in the Output panel
4. When complete, review the Error List — resolve any unexpected errors before proceeding to data migration

> **Known SSMA behavior:** Two benign issues may appear during sync and can be safely worked around. First, SSMA may report an identity column error on some tables — this is a known bug; the tables are created correctly, so verify in SSMS and proceed. Second, triggers may fail to deploy due to ordering constraints — always create triggers manually in SSMS after sync completes (covered in Step 2.3).

### Step 2.3 — Create triggers manually in SSMS

**What and why:** Due to SSMA's known trigger deployment ordering issue, triggers should always be created directly in SSMS after the rest of the schema is in place. Copy the trigger T-SQL from SSMA's converted script editor and run it in SSMS once the parent tables are confirmed present.

```sql
-- General pattern — replace with your actual converted trigger body
CREATE TRIGGER [your_schema].[trigger_name]
ON [your_schema].[table_name]
AFTER UPDATE  -- or INSERT, DELETE
AS
BEGIN
    -- trigger logic here
END;
GO
```

### Step 2.4 — Validate the deployed schema

**What and why:** Before migrating any data, confirm every object from the DB2 source was deployed to Azure SQL. Compare the counts returned here against the object inventory captured in Step 1.1 — they must match. Any missing object must be resolved before proceeding.

Run in SSMS with the target database selected:

```sql
-- Tables and views
SELECT TABLE_NAME, TABLE_TYPE
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'YOUR_SCHEMA'
ORDER BY TABLE_TYPE, TABLE_NAME;

-- Stored procedures
SELECT ROUTINE_NAME, ROUTINE_TYPE
FROM INFORMATION_SCHEMA.ROUTINES
WHERE ROUTINE_SCHEMA = 'YOUR_SCHEMA';

-- Triggers
SELECT t.name AS TriggerName, OBJECT_NAME(t.parent_id) AS TableName, t.is_disabled
FROM sys.triggers t
WHERE OBJECT_SCHEMA_NAME(t.parent_id) = 'YOUR_SCHEMA';
```

All object counts must match the Step 1.1 baseline and all triggers must show `is_disabled = 0` before proceeding to data migration.

---

## Phase 3 — Data Migration

**Goal:** Move all data from DB2 source tables to the corresponding Azure SQL target tables with full row-count and spot-check validation.

### Choose your migration approach

Select the tool that matches your environment and tolerance for downtime. **Complete only one of the three options below** — they are mutually exclusive paths to the same outcome.

| | Option A — SSMA | Option B — Azure DMS | Option C — Striim |
|---|---|---|---|
| **Best for** | POC, dev/test, small workloads | Production with a maintenance window | Production with near-zero downtime |
| **CDC support** | No — full load only | Yes (Premium tier) | Yes — real-time log-based |
| **Cost** | Free | Azure-billed managed service | Commercial license required |
| **Complexity** | Low — reuses existing SSMA setup | Medium — portal-driven setup | Higher — pipeline configuration |

> **Not sure which to pick?** If the DB2 source can be taken offline during migration, Option A or B is sufficient. If DB2 must remain live and writes continue during the migration window, use Option B (Online mode) or Option C.

### Step 3.1 — Capture the DB2 row count baseline

Before migrating, record exact row counts from DB2 for every table. These numbers are your validation targets in Phase 4 — if the counts on Azure SQL do not match after migration, rows were lost. Everyone completes this step regardless of which migration option is chosen.

```sql
-- Run on DB2 source immediately before starting migration
SELECT TABNAME, CARD AS EstimatedRows
FROM SYSCAT.TABLES
WHERE TABSCHEMA = 'YOUR_SCHEMA'
ORDER BY TABNAME;
```

> `CARD` is DB2's catalog statistic and may be stale. For guaranteed accuracy run `SELECT COUNT(*) FROM schema.tablename` per table if the database is actively receiving inserts.

---

### Option A — Migrate data using SSMA (full-load)

**What and why:** SSMA includes a built-in bulk data copy feature that reads every row from each DB2 table and inserts it into the corresponding Azure SQL table. It reuses the connections already configured during the assessment phase — no additional tooling is needed. This is the simplest and fastest path for POC environments and non-production migrations. It performs a one-time full load with no ongoing replication.

**How to run it:**

1. In SSMA, select your schema in the DB2 explorer, right-click → **Migrate Data**
2. Confirm the source and target connections shown in the dialog, then click **Migrate**
3. The Output panel logs progress per table — rows read, rows inserted, and any errors
4. When complete, note the row counts — compare against the Step 3.1 baseline in Phase 4

---

### Option B — Migrate data using Azure Database Migration Service

**What and why:** Azure DMS is a fully managed Azure service that connects directly to both DB2 and Azure SQL. In Online mode it performs an initial full load then switches to CDC, continuously streaming every transaction from DB2 into Azure SQL until you explicitly initiate cutover. No Windows intermediary is required. This is the right choice for production migrations where the DB2 source must remain live throughout the migration window.

**How to run it:**

1. In the Azure portal, search **Azure Database Migration Service** → Create — select the Premium tier (required for CDC) and place it in the same region as your Azure SQL target
2. Create a new migration project: source type **IBM DB2**, target type **Azure SQL MI** or **Azure SQL Database**
3. Configure the source DB2 connection (hostname, port, database, credentials with log read authority) and the target Azure SQL connection
4. Select the schemas and tables to include, then choose the migration mode:
   - **Online** — full load then continuous CDC; near-zero downtime, manual cutover when lag = 0
   - **Offline** — full load only; DB2 must be quiesced during migration
5. Start the migration — DMS handles the full load and CDC transition automatically
6. When the DMS dashboard shows all tables as "Ready to cutover" and lag = 0, initiate cutover from the portal

---

### Option C — Migrate data using Striim (real-time CDC replication)

**What and why:** Striim is a commercial real-time integration platform that reads directly from the DB2 transaction log. It performs an initial full load to seed the Azure SQL target, then seamlessly transitions to streaming CDC — continuously applying every insert, update, and delete with sub-second latency. Cutover becomes a simple connection string redirect with no data loss risk. Striim is the right choice for large databases, high transaction volumes, or situations where even a brief maintenance window is not acceptable.

**How to run it:**

1. Deploy Striim on Azure via the Marketplace (VM image or managed SaaS) — size based on DB2 transaction volume (8 vCores / 32 GB is a reasonable starting point for medium workloads)
2. Create a pipeline with a **DB2Reader** source (pointing at the DB2 transaction log — requires `LOGARCHMETH1` or equivalent log access configured on DB2) and a **SQLServerWriter** target pointing at Azure SQL
3. Start the pipeline in **Initial Load** mode — Striim bulk-copies all existing rows into Azure SQL, then automatically transitions to real-time CDC replication
4. Monitor the pipeline lag metric in the Striim dashboard — it should stabilize near zero, indicating Azure SQL is current with DB2
5. When ready to cut over: stop writes to DB2, confirm lag = 0 in Striim, then redirect application connection strings to Azure SQL — Striim can remain running as a safety net until the DB2 source is formally decommissioned

---

## Phase 4 — Validation

**Goal:** Confirm data integrity and functional correctness on the Azure SQL target before decommissioning DB2.

### Step 4.1 — Row count comparison

**What and why:** This is the first and most fundamental check — it tells you whether all the data arrived. By comparing the row count for every table on Azure SQL against the baseline captured in Step 3.1, you are confirming that the migration engine did not silently drop, skip, or truncate any records. A mismatch here is a blocker — do not proceed to cutover until every table matches.

**Expected outcome:** Every table on Azure SQL returns the same count as the DB2 baseline. No exceptions.

Run on the Azure SQL target and compare against Step 3.1:

```sql
-- Replace with your actual table list
SELECT 'TABLE_1' AS TableName, COUNT(*) AS RecordCount FROM [schema].[TABLE_1]
UNION ALL
SELECT 'TABLE_2',              COUNT(*) FROM [schema].[TABLE_2]
UNION ALL
SELECT 'TABLE_3',              COUNT(*) FROM [schema].[TABLE_3]
ORDER BY TableName;
```

If any count is off, re-run the migration for that table or investigate truncation errors in the migration tool's output log before continuing.

### Step 4.2 — Spot-check data values

**What and why:** Row counts confirm quantity but not quality — a table can have the right number of rows while still containing corrupted or mis-converted values. This step samples actual records side-by-side between DB2 and Azure SQL to catch data type conversion issues that are easy to overlook: dates shifted by a timezone offset, decimal precision quietly truncated, or fixed-length strings padded differently. These are the kinds of errors that only surface in production after cutover if not caught here.

**Expected outcome:** Sampled records on Azure SQL are byte-for-byte identical to their DB2 counterparts — same primary keys, same field values, same precision.

```sql
-- On Azure SQL target
SELECT TOP 10 * FROM [schema].[TABLE_NAME] ORDER BY 1;

-- Equivalent on DB2 source — run and compare manually
SELECT * FROM schema.TABLE_NAME ORDER BY 1 FETCH FIRST 10 ROWS ONLY;
```

Pay particular attention to:
- `DATE` / `TIMESTAMP` fields — DB2 and SQL Server handle timezone differently
- `DECIMAL` fields — confirm precision and scale are preserved
- `CHAR` fields with trailing spaces — DB2 pads fixed-length chars; SQL Server may not

### Step 4.3 — Functional validation

**What and why:** Even with correct data in place, the application still needs to work. Stored procedures, views, and joins may reference objects or use syntax that behaved slightly differently on DB2. This step exercises the database logic — not just the data — to confirm that the objects deployed in Phase 2 execute correctly and return the results the application expects. Think of this as a smoke test for the database layer before any application teams reconnect.

**Expected outcome:** All stored procedures execute without error, all views return results, and cross-table joins produce consistent output — matching what the same queries returned on DB2.

```sql
-- Test a stored procedure
EXEC [schema].[procedure_name] @param1 = value;

-- Test a view
SELECT TOP 100 * FROM [schema].[view_name];

-- Test a cross-table join to confirm foreign key integrity
SELECT a.*, b.*
FROM [schema].[TABLE_A] a
JOIN [schema].[TABLE_B] b ON a.id = b.foreign_key_id
WHERE a.id = 1;
```

### Step 4.4 — Index and performance baseline

**What and why:** A database with correct data but poor performance is not ready for production. When data is bulk-loaded, SQL Server's internal statistics — which the query optimizer relies on to choose efficient execution plans — can be stale or missing entirely. This step refreshes those statistics and surfaces any indexes that exist on DB2 but were not carried over, or new ones that Azure SQL recommends based on query patterns. Addressing this before go-live prevents performance regressions that would otherwise only be discovered under production load.

**Expected outcome:** Statistics are current, and any high-impact missing indexes flagged by the query are reviewed and either created or consciously accepted as out of scope.

```sql
-- Refresh statistics on all tables in the target database
EXEC sp_updatestats;

-- Identify high-impact missing indexes based on query patterns observed so far
SELECT TOP 10
    migs.avg_user_impact,
    mid.statement AS TableName,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns
FROM sys.dm_db_missing_index_group_stats migs
JOIN sys.dm_db_missing_index_groups mig ON migs.group_handle = mig.index_group_handle
JOIN sys.dm_db_missing_index_details mid ON mig.index_handle = mid.index_handle
ORDER BY migs.avg_user_impact DESC;
```

> Run representative application queries against the Azure SQL target before checking for missing indexes — the DMV only reports indexes for queries SQL Server has already seen and analyzed.

---

## Phase 5 — Cutover

**Goal:** Safely redirect the application from DB2 to Azure SQL with a clear go/no-go decision, a defined rollback escape route, and a monitoring window before the DB2 source is decommissioned.

> **Key principle:** DB2 should not be decommissioned the moment cutover completes. Set it to read-only and keep it available for a defined hold period — typically 5–10 business days — as a safety net. Decommission only after the application has been running stably on Azure SQL with no rollback events.

### Step 5.1 — Go/No-Go gate

**What and why:** Before touching anything in production, the team needs a formal confirmation that every phase of the migration was completed and passed. This is the decision point — not a formality. If any item below cannot be checked, the cutover is postponed until it can be.

**Expected outcome:** All boxes checked. The team has explicit sign-off to proceed.

- [ ] Phase 1 assessment report reviewed and all Red-flagged objects resolved
- [ ] All DB2 schema objects deployed to Azure SQL (Step 2.4 object counts match)
- [ ] All triggers created and enabled on Azure SQL
- [ ] Row counts match for every table (Step 4.1)
- [ ] Spot-check data values passed for key tables (Step 4.2)
- [ ] All stored procedures and views execute correctly on Azure SQL (Step 4.3)
- [ ] Statistics refreshed and high-impact missing indexes addressed (Step 4.4)
- [ ] Azure SQL backup confirmed
- [ ] Rollback plan documented and communicated to the team (Step 5.3)
- [ ] Maintenance window scheduled and stakeholders notified

### Step 5.2 — Cutover sequence

**What and why:** Cutover is the moment the application stops writing to DB2 and begins writing to Azure SQL. The sequence below minimises the window during which neither system is authoritative. The exact steps vary slightly depending on which data migration option was used — SSMA (full-load) requires a clean stop of DB2 writes before cutover, while DMS and Striim (CDC) allow writes to continue right up until the connection string is switched.

**Expected outcome:** The application is fully connected to Azure SQL, passes a smoke test under real conditions, and DB2 is set to read-only. No data is written to DB2 after this point.

**If you used Option A (SSMA — full-load):**

1. Schedule a maintenance window — application writes must be stopped during cutover
2. Stop the application or set it to maintenance mode to prevent any new writes to DB2
3. Run a final row count check on both DB2 and Azure SQL to confirm they still match
4. Update all application connection strings to point to the Azure SQL endpoint
5. Restart the application and run a smoke test — confirm core workflows complete successfully
6. Set the DB2 instance to read-only: `UPDATE DBM CFG USING UPDATE_DBM_CFG NO` or restrict access at the network level
7. Declare cutover complete and begin the post-cutover monitoring window (Step 5.4)

**If you used Option B (Azure DMS) or Option C (Striim — CDC):**

1. Confirm replication lag is at or near zero in the DMS dashboard or Striim pipeline monitor
2. Stop the application or set it to maintenance mode (this window is much shorter — seconds to minutes rather than the full migration duration)
3. Confirm lag has reached exactly zero — all in-flight transactions from DB2 have been applied to Azure SQL
4. Update all application connection strings to point to the Azure SQL endpoint
5. Restart the application and run a smoke test — confirm core workflows complete successfully
6. Stop the DMS migration task or Striim pipeline (replication is no longer needed)
7. Set the DB2 instance to read-only
8. Declare cutover complete and begin the post-cutover monitoring window (Step 5.4)

### Step 5.3 — Rollback plan

**What and why:** Define the rollback plan *before* cutover begins — not after something goes wrong. If a critical issue surfaces post-cutover that cannot be resolved quickly, the team needs a clear, pre-agreed procedure to revert to DB2 without scrambling. The fact that DB2 is in read-only (not decommissioned) makes this possible.

**Expected outcome:** Every team member knows the rollback threshold and the exact steps to revert before cutover is initiated.

Key decisions to make and document in advance:

| Decision | Guidance |
|---|---|
| **Rollback threshold** | Define a time limit — e.g., if a critical blocker is not resolved within 2 hours of cutover, rollback is initiated |
| **Rollback trigger** | Who has authority to call a rollback? Define one decision maker |
| **DB2 hold period** | How many business days will DB2 remain in read-only before decommission? (5–10 days is typical) |
| **Connection string revert** | Keep the original DB2 connection strings documented and accessible — rollback is as simple as swapping them back |

**Rollback steps if needed:**

1. Stop the application
2. Revert all connection strings to the original DB2 endpoint
3. Remove the read-only restriction on DB2
4. Restart the application and confirm it is writing to DB2 again
5. Investigate and resolve the root cause on Azure SQL before attempting cutover again

> Any data written to Azure SQL between cutover and rollback will need to be reconciled manually — this is why the rollback window must be short and the threshold clearly defined.

### Step 5.4 — Post-cutover monitoring

**What and why:** The first 24–72 hours after cutover are the highest-risk period. Application behaviour under real production load may surface issues that were not visible in validation — query plans that perform differently at scale, edge-case data values not covered by spot-checks, or application code paths that were not exercised during smoke testing. Active monitoring during this window allows the team to catch and address issues while DB2 is still available as a fallback.

**Expected outcome:** After the monitoring window passes with no rollback events and no unresolved critical issues, the team can formally schedule DB2 decommission with confidence.

What to monitor:

| Area | What to watch |
|---|---|
| **Query performance** | Compare execution times for key queries against the DB2 baseline — flag anything significantly slower |
| **Error rates** | Monitor application logs for database errors, timeouts, or connection failures |
| **SQL dialect issues** | Watch for any runtime errors from T-SQL syntax differences not caught during functional validation |
| **Storage and DTU/vCore utilisation** | Confirm the Azure SQL tier is appropriately sized under real load — scale up if sustained utilisation exceeds 80% |
| **Replication (if CDC was used)** | Confirm the DMS task or Striim pipeline is stopped and no stale replication is running |

After the monitoring window closes with a stable system, proceed to formally decommission the DB2 source per your organisation's infrastructure retirement process.

---

## Post-Migration Checklist

Before decommissioning the DB2 source:

- [ ] All object counts match between source and target
- [ ] Row counts match for every table
- [ ] Spot-check data values pass for key tables
- [ ] All stored procedures execute without error
- [ ] All views return expected results
- [ ] All triggers fire correctly on test updates
- [ ] Application connection strings updated to point to Azure SQL
- [ ] Application smoke test passed in the new environment
- [ ] Backup of Azure SQL target confirmed
- [ ] DB2 instance set to read-only (do not decommission until post-go-live monitoring is complete)

---

## Common Issues and Fixes

| Symptom | Likely Cause | Fix |
|---|---|---|
| SSMA "user doesn't have authority to access the host" | DB2 OS password not set or doesn't match | Run `sudo passwd <db2_instance_owner>` on the DB2 host and reset the password; retry |
| SSMA "Cannot update identity column" during sync | Known SSMA bug — post-create identity column operation | Verify tables were created correctly in SSMS; proceed with manual trigger creation |
| SSMA "all objects are equal" on empty target | SSMA's converted model was lost after reconnect | Re-run Convert Schema, then Synchronize again |
| SSMA "AccessToken cannot be null" connecting to SQL MI | MFA token not returned after browser sign-in | Reconnect to SQL MI in SSMA toolbar to trigger a fresh MFA prompt |
| Trigger creation fails with "object does not exist" | Trigger was deployed before its parent table | Create the trigger manually in SSMS after all tables are confirmed present |
| Row count mismatch after migration | Rows inserted to DB2 during SSMA full-load | Re-run migration or use DMS/Striim with CDC for production |
| `Incorrect syntax near the keyword 'RowCount'` | `RowCount` is a reserved word in SQL Server | Rename the alias to `RecordCount` or wrap in brackets `[RowCount]` |
| DB2 `FETCH FIRST N ROWS ONLY` fails on Azure SQL | DB2 syntax not supported in T-SQL | Replace with `SELECT TOP N` or `ORDER BY ... OFFSET 0 ROWS FETCH NEXT N ROWS ONLY` |
