# Developer User Guide — Fabric Engineering Framework

## Table of Contents
1. [Prerequisites](#1-prerequisites)
2. [Repository Structure](#2-repository-structure)
3. [Initial Setup](#3-initial-setup) — including [Naming Conventions](#33-rename-artifacts-to-match-project-naming-convention), [Fabric API Connection](#35-create-the-fabric-api-connection) and [Artifact Cleanup](#36-remove-example-and-unused-artifacts)
4. [Configuration Approach Decision](#4-configuration-approach-decision)
5. [Option 1 — Database Config Tables Setup](#5-option-1--database-config-tables-setup)
6. [Option 2 — YAML Config Files Setup](#6-option-2--yaml-config-files-setup)
7. [YAML Orchestration Modes](#7-yaml-orchestration-modes)
8. [Async Logging Architecture](#8-async-logging-architecture)
9. [YAML Configuration Schema Reference](#9-yaml-configuration-schema-reference)
10. [Adding a New Data Source (YAML mode)](#10-adding-a-new-data-source-yaml-mode)
11. [Resume After Failure](#11-resume-after-failure)
12. [Cross-Workspace Execution](#12-cross-workspace-execution)
13. [Variable Library](#13-variable-library)
14. [Notebook Development Patterns](#14-notebook-development-patterns)
15. [CI/CD and Deployment](#15-cicd-and-deployment)
16. [Monitoring and Troubleshooting](#16-monitoring-and-troubleshooting)
17. [Common Pitfalls](#17-common-pitfalls)
18. [Landing Zone and Bronze Change Detection](#18-landing-zone-and-bronze-change-detection)
19. [Silver History (SCD2) Load Type](#19-silver-history-scd2-load-type)

---

## 1. Prerequisites

| Requirement | Notes |
|---|---|
| Microsoft Fabric workspace (F2 SKU minimum) | F4+ recommended for production workloads |
| Git repository | Azure DevOps or GitHub (see note below) |
| Fabric workspace admin permissions | Required for source control sync and shortcut creation |
| Fabric API connection (`cn_Fabric`) | Required by pipelines that call Fabric REST API (see [Section 3.5](#35-create-the-fabric-api-connection)) |
| Eventhouse (YAML option) | `eh_<projectName>_01` — for centralized logging |
| Fabric Warehouse (Database option) | `dw_<projectName>_01` — for configuration tables |
| Python 3.11+ (local development) | For YAML validation scripts |

> **Git provider**: Fabric source control supports both **Azure DevOps** and **GitHub**. However, the CI/CD pipeline files in the `CICD/` folder are written for **Azure DevOps Pipelines** only. If you use GitHub, you will need to rewrite the CI/CD workflows as GitHub Actions manually — the provided YAML pipeline files will not work as-is with GitHub.

---

## 2. Repository Structure

```
├── CHANGELOG.md
├── README.md
├── docs/
│   └── user-guide.md                   ← this file
├── CICD/
│   ├── azure-pipelines.yml             Main pipeline trigger
│   ├── deploy-configFiles.yml          Deploy YAML config to OneLake
│   └── deploy-cross-environment.yml    Cross-environment Fabric artifact deployment
├── ConfigFiles/
│   ├── README.md                       Schema documentation
│   ├── environments/
│   │   ├── dev/
│   │   │   ├── framework.yml           Model YAML config (dev)
│   │   │   └── inferredMembersConfig.yml
│   │   └── test/
│   └── scripts/
│       ├── config-schema.json          JSON Schema for YAML validation
│       ├── validate_config.py          Python validation script
│       └── deploy-configOnelake.ps1    Deployment helper
├── Notebooks/
│   ├── 99_Administration/
│   │   └── CommonFunctions.Notebook/   Shared library — always %run first
│   ├── 98_Yaml_Orchestrator/           YAML orchestration notebooks
│   ├── 00_Bronze/                      nb_promote_landing_to_bronze — see Section 18
│   ├── 10_Silver/                      Silver layer notebooks
│   ├── 20_Gold/                        Gold layer notebooks
│   └── 90_Utilities/                   Utility notebooks
├── Pipelines/                          Database-config approach pipelines
├── Pipelines_NoSQL/                    YAML-config approach pipelines
├── Lakehouses/                         Landing, Bronze, Silver, Gold lakehouse definitions
├── Eventhouses/                        KQL database and querysets
├── Variables/
│   └── vl_lz036_01.VariableLibrary/  Fabric IDs per environment
└── Warehouses/                         Config warehouse (Database option)
```

---

## 3. Initial Setup

### 3.1 Clone and Connect to Fabric Source Control

1. Clone this repository to your **Azure DevOps** or **GitHub** project.
2. In the Fabric workspace, go to **Workspace Settings → Git integration**.
3. Connect to the repo and branch (e.g., `dev`).
4. Perform an **Update all** sync to import all artifacts into the workspace.

> **CI/CD compatibility**: The pipeline files in `CICD/` are written for Azure DevOps Pipelines. If your project uses GitHub, you must rewrite them as GitHub Actions workflows.

> After the initial sync, some artifacts may show connection errors because connection IDs in pipelines reference the original environment. See [Section 3.4](#34-fix-pipeline-connections).

> Artifacts in this repository have already been renamed from the `modernBI-lz036` template prefix to `lz036` (see [Section 3.3](#33-rename-artifacts-to-match-project-naming-convention)). Complete [Section 3.5](#35-create-the-fabric-api-connection) to create the required Fabric API connection after the first sync.

> **Not all artifacts are required.** This repository contains the full framework template, including artifacts for both configuration approaches and several example/demo objects. You are free — and encouraged — to delete any artifact that is not needed for your chosen approach or project scope. See [Section 3.6](#36-remove-example-and-unused-artifacts) for a deletion checklist.

### 3.2 Create and Populate the Variable Library

The Variable Library `vl_lz036_01` stores all environment-specific IDs. Update it with the correct values for your workspace **before** running anything:

| Variable | Description | Where to find it |
|---|---|---|
| `workspaceId` | Fabric workspace GUID | Workspace Settings |
| `landing_lakehouseId` | Landing lakehouse GUID (see [Section 18](#18-landing-zone-and-bronze-change-detection)) | Lakehouse Settings |
| `bronze_lakehouseId` | Bronze lakehouse GUID | Lakehouse Settings |
| `silver_lakehouseId` | Silver lakehouse GUID | Lakehouse Settings |
| `gold_lakehouseId` | Gold lakehouse GUID | Lakehouse Settings |
| `warehouseId` | Warehouse GUID (Database option) | Warehouse Settings |
| `warehouseConnectionString` | Warehouse SQL endpoint | Warehouse Settings → SQL endpoint |
| `eventhouseId` | Eventhouse GUID (YAML option) | Eventhouse Settings |
| `eventhouseName` | Eventhouse display name | e.g., `eh_lz036_01` |
| `eventhouseDatabase` | KQL database name | e.g., `eh_lz036_01` |
| `eventhouseWorkspaceId` | Workspace GUID where Eventhouse lives | Same as `workspaceId` unless cross-workspace |

Edit `Variables/vl_lz036_01.VariableLibrary/variables.json` and sync to Fabric, or update the values directly in the Fabric portal.

### 3.3 Rename Artifacts to Match Project Naming Convention

All artifacts in this repository have been renamed from the `modernBI-lz036` template prefix to the `lz036` project name. The table below documents the original template names and the names used in this repository.

**Rename the following categories:**

| Category | Template name example | Rename to example |
|---|---|---|
| Lakehouses | `lh_lz036_bronze_01` | `lh_<projectName>_bronze_01` |
| Eventhouse | `eh_lz036_01` | `eh_<projectName>_01` |
| KQL Database | `eh_lz036_01` | `eh_<projectName>_01` |
| Warehouse | `dw_lz036_01` | `dw_<projectName>_01` |
| Variable Library | `vl_lz036_01` | `vl_lz036_01` |
| Pipelines | `pl_lz036_master` | `pl_<projectName>_master` |
| Notebooks (if renamed) | Keep generic names where possible (e.g., `nb_load_silver`) | — |
| YAML config values | `lh_lz036_bronze_01` references inside YAML files | Update to match new lakehouse names |
| Variable Library values | `eventhouseName`, `eventhouseDatabase` inside `variables.json` | Update to match renamed Eventhouse |

**Recommended approach — rename in the repository before the first sync:**

Each Fabric artifact in the repo consists of a **folder** (e.g., `lh_lz036_bronze_01.Lakehouse/`) and a **`.platform` file** inside it containing the `displayName`. Cross-artifact references inside pipelines use the stable `logicalId` — not the display name — so renaming in the repo is safe and does not break any cross-artifact links. Renaming before the first sync means the workspace never sees the template names.

1. In VS Code, use **Find in Files** (`Ctrl+Shift+F`) with the search term `modernBI-lz036` to locate every occurrence across folder names, `.platform` files, YAML configs, and pipeline JSON.
2. Rename each artifact **folder** to the new name (e.g., `lh_lz036_bronze_01.Lakehouse` → `lh_<projectName>_bronze_01.Lakehouse`).
3. Inside each renamed folder, open the `.platform` file and update the `displayName` to match the new folder name. The folder name and `displayName` must stay in sync — Fabric enforces this at sync time.
4. Update YAML config files: replace all `destinationSystemName`, `destinationDatabase`, and `sourceSystemName` values that reference the old lakehouse or eventhouse names.
5. Update `variables.json` in the Variable Library: update `eventhouseName` and `eventhouseDatabase` to the new names.
6. Commit all changes, then perform the first **Update all** sync from Fabric.

> **Do not change the `logicalId`** in any `.platform` file. It is the stable identifier Fabric uses to track and link artifacts across environments — changing it will break cross-artifact references.

### 3.4 Fix Pipeline Connections

After syncing, open each pipeline that connects to an external source and update the linked service / connection to a valid connection in the target environment. This must be done before the first run.

### 3.5 Create the Fabric API Connection

Several pipelines use **Web activities** or **Invoke Pipeline activities** that call the Fabric REST API. These activities rely on a connection named **`cn_Fabric`** to authenticate against the API.

**Create the connection:**
1. In the Fabric workspace, go to **Settings → Manage connections and gateways**.
2. Create a new connection of type **Web** (or **Anonymous** depending on the authentication method supported by your tenant).
3. Set the connection name to **`cn_Fabric`** (must match exactly — pipelines reference this name).
4. Set the URL to `https://api.fabric.microsoft.com`.
5. Set the authentication kind to the method available in your tenant (typically **Organizational account** / **Service Principal**).

> If you rename the connection, you must update all pipeline activities that reference `cn_Fabric` to use the new name.

### 3.6 Remove Example and Unused Artifacts

This repository ships with demo artifacts that illustrate how the framework works but are **not intended for use in a real project**. Remove anything that does not apply to your target project before your first deployment to avoid confusion.

**Always remove — these are examples only:**

| Artifact | Reason |
|---|---|
| `pl_copyFrom_adventureWorks2022_kql` | Example ingestion pipeline for the AdventureWorks 2022 demo database. Replace with your own source pipelines. |
| `pl_copyFrom_adventureWorks2022_kql_yaml` | Same as above — the YAML-mode variant. |
| `ConfigFiles/environments/dev/framework.yml` | Example YAML configuration file using AdventureWorks. Use it as a reference, then replace its contents with your own model configuration. |
| `ConfigFiles/environments/dev/framework_xl.yml` | Extended example config. Reference only — delete or replace. |
| `ConfigFiles/environments/dev/inferredMembersConfig.yml` | Example inferred members config. Adapt values to your project's data types and surrogate key conventions before deploying. |

**Remove if not using the Database Config Tables approach (Option 1):**
- `Pipelines/` folder contents (all traditional pipeline artifacts)
- `Warehouses/dw_lz036_01.Warehouse/`

**Remove if not using the YAML Config Files approach (Option 2):**
- `Pipelines_NoSQL/` folder contents
- `Eventhouses/` folder contents
- `Notebooks/98_Yaml_Orchestrator/` folder

**Remove if not using cross-workspace execution:**
- No specific deletions needed — the `artifactWorkspace` property is optional and has no runtime cost when omitted.

---

## 4. Configuration Approach Decision

| Factor | Database Config Tables | YAML Config Files |
|---|---|---|
| Version control for config | No | **Yes** — YAML files in Git |
| Config management | SQL queries | Text editor / IDE |
| Shortcuts required | **Yes** | No |
| Logging | Warehouse tables | Eventhouse (async file-based) |
| Resume after failure | No | **Yes** |
| Recommended for new projects | No | **Yes** |

For all new projects, use **Option 2 (YAML Config Files)** unless there is a specific reason to use warehouse tables.

---

## 5. Option 1 — Database Config Tables Setup

### 5.1 Create Shortcuts

In the **Silver lakehouse**, create shortcuts (schema: `config`) pointing to these warehouse tables:
- `admin.processDatesConfig`
- `admin.silverGoldConfig`
- `admin.silverGoldDependency`

In the **Silver lakehouse**, create a shortcut (schema: `log`) pointing to:
- `log.copyDataLog`

In the **Gold lakehouse**, create shortcuts (schema: `log`) pointing to:
- `admin.inferredMembersConfig`
- `admin.processDatesConfig`
- `admin.silverGoldConfig`
- `admin.silverGoldDependency`

### 5.2 Populate Config Tables

Connect to `dw_lz036_01` and insert rows into:
- `admin.copyDataConfig` — one row per source table/file to extract
- `admin.silverGoldConfig` — one row per Silver or Gold artifact to execute
- `admin.silverGoldDependency` — dependency pairs between artifacts

Refer to the column descriptions in [README.md](../README.md#config-database) for the full schema.

### 5.3 Execute

Run `pl_run_lz036_master` to fetch Fabric IDs, then pass them to `pl_lz036_master`.

---

## 6. Option 2 — YAML Config Files Setup

### 6.1 Create the KQL Schema

The KQL schema is defined in `Eventhouses/eh_lz036_01.KQLDatabase/DatabaseSchema.kql` and is applied automatically when Fabric syncs the repository for the first time. The following objects are created:
- `processExecutionLog` — execution log table
- `copyDataLog` — ingestion log table
- `datesConfig` — delta load watermark table
- `LastFailedBatchProcesses` — function to retrieve the last failed batch
- `lastDatesConfig` — materialised view for the latest watermark per table

No manual KQL execution is required after renaming the Eventhouse and performing the initial sync.

### 6.2 Create the YAML Configuration File

> **The files in `ConfigFiles/environments/dev/` are examples only.** They reference the AdventureWorks 2022 demo database and the `modernBI-lz036` template artifact names. Do not deploy them to a real project workspace without first replacing their contents with your own model configuration.

1. Use `ConfigFiles/environments/dev/framework.yml` as a **reference** for the YAML structure.
2. Create a new YAML file for your model (e.g., `ConfigFiles/environments/dev/<yourModel>.yml`).
3. Add your model's `raw`, `silver`, and `gold` sections (see [Section 9](#9-yaml-configuration-schema-reference)).
4. Validate locally:
   ```bash
   python ConfigFiles/scripts/validate_config.py ConfigFiles/environments/dev/<yourModel>.yml
   ```

### 6.3 Deploy the YAML File to OneLake

**Manually (first time or ad-hoc):**

Upload the YAML file to the Bronze lakehouse at:
```
Files/config/<model>.yml
```

**Via CI/CD (recommended):**

Push to the `ConfigFiles/environments/dev/` folder. The `deploy-configFiles.yml` **Azure DevOps** pipeline detects changes and deploys automatically to the target environment lakehouse. See [Section 15](#15-cicd-and-deployment).

> **GitHub users**: The provided CI/CD pipeline files do not work with GitHub Actions. You will need to create equivalent GitHub Actions workflows to automate YAML config deployment.

### 6.4 Execute

Run `pl_master_YamlOrchestrator` (or `pl_master`) which calls `nb_yaml_orchestrator` (or `nb_yaml_orchestrator_batch`) with the `model` parameter set to your model name.

Parameters accepted by the orchestrator notebooks:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `model` | string | `"framework"` | Model name — must match the `model:` field in the YAML file |
| `modelReset` | bool | `True` | `True` = full run. `False` = resume from last failure (see [Section 11](#11-resume-after-failure)) |
| `modelDate` | string | `None` | Override delta logic to reprocess a specific date (`YYYY-MM-DD`) |
| `context` | string | `""` | Free-text context label attached to all log entries for this run |

---

## 7. YAML Orchestration Modes

### nb_yaml_orchestrator — Standard Mode

The orchestrator reads the YAML file and creates **one DAG activity per active entry** across all sections (bronze/silver/gold). Each bronze entry triggers its configured pipeline individually.

**Execution flow:**

```
nb_yaml_orchestrator
  ├── bronze_Person         (→ pl_copyFrom_adventureWorks2022_kql_yaml)
  ├── bronze_SalesOrderDetail (→ pl_copyFrom_adventureWorks2022_kql_yaml)
  ├── silver_Person         (depends on: bronze_Person, bronze_SalesPerson)
  ├── silver_Store          (depends on: bronze_Store)
  ├── gold_dim_person       (depends on: silver_Person)
  └── nb_log_file_monitor     (no dependencies — runs concurrently)
```

Silver activities unblock individually as their specific bronze dependencies complete.

### nb_yaml_orchestrator_batch — Batch Bronze Mode

All active bronze entries are folded into a **single `bronze_batch` DAG activity**. The pipeline `pl_lz036_copyFromSource_kql` receives the full list of extraction configs as an `items` array and executes all ingestion internally (e.g., using ForEach activities).

**Execution flow:**

```
nb_yaml_orchestrator_batch
  ├── bronze_batch          (→ pl_lz036_copyFromSource_kql
  │                            with ALL bronze entries as items[])
  ├── silver_Person         (depends on: bronze_batch)
  ├── silver_Store          (depends on: bronze_batch)
  ├── gold_dim_person       (depends on: silver_Person)
  └── nb_log_file_monitor     (no dependencies — runs concurrently)
```

Silver activities can only start after the entire bronze batch completes.

### When to Use Each Mode

| Scenario | Recommended Mode |
|---|---|
| F2/F4 Fabric SKU | `nb_yaml_orchestrator_batch` |
| > 10 bronze source tables | `nb_yaml_orchestrator_batch` |
| Bronze extractions have very unequal durations | `nb_yaml_orchestrator` |
| You need silver to start before all bronze finishes | `nb_yaml_orchestrator` |
| First implementation / default choice | `nb_yaml_orchestrator_batch` |

---

## 8. Async Logging Architecture

### The Problem with Synchronous KQL Logging

Previous implementations wrote each log entry directly to the Eventhouse KQL database during notebook execution. Under concurrent workloads (multiple notebooks running in parallel) this caused:
- KQL throttling errors (HTTP 429)
- Increased CU consumption for logging overhead
- Delayed pipeline and notebook execution while waiting for KQL acknowledgement

### The Solution: File-Based Async Logging

Each notebook writes log entries as small JSON files to the Bronze lakehouse file system (`Files/logs/`). A dedicated monitor notebook (`nb_log_file_monitor`) runs concurrently and batch-ingests these files into KQL at regular intervals.

### Log Flow

1. **Activity starts** → `nb_wrapper` calls `writeLogStarted()` from `CustomLogging`
2. **`writeLogStarted()`** → writes a JSON file to `Files/logs/processExecutionLog/<name>_<timestamp>.json`
3. **Activity completes** → `nb_wrapper` calls `writeLogFinishedSuccess()` or `writeLogFinishedError()`
4. **Completion** → `nb_wrapper` writes a `.done` sentinel file to `Files/logs/.done/<processName>`
5. **`nb_log_file_monitor`** (running in parallel) → scans log folders every 60 s, reads new JSON files, batch-ingests to KQL, archives processed files
6. **Monitor exits** when it detects all `.done` sentinels (count = total DAG activities) plus a 5-minute drain period to capture any late-arriving logs

### Log File Schema — processExecutionLog

```json
{
  "model": "framework",
  "batchExecution": "uuid-v4",
  "workspaceId": "...",
  "processName": "silver_Person",
  "subProcessName": "",
  "processInternalId": "silver_Person",
  "processType": "notebook",
  "subProcessType": "",
  "status": "Succeeded",
  "errorMessage": "",
  "timestamp": "2026-05-08T10:30:00.123456",
  "context": "daily-run"
}
```

### nb_log_file_monitor Configuration Parameters

These are set as parameters in the DAG activity args (controlled by the orchestrator):

| Parameter | Default | Description |
|---|---|---|
| `expected_count` | set by orchestrator | Number of `.done` sentinels expected (= number of DAG activities) |
| `check_interval_seconds` | `60` | Polling interval in seconds |
| `batch_size` | `50` | Files to ingest per KQL write operation |
| `drain_duration_seconds` | `300` | Time to keep running after all sentinels received |
| `idle_timeout_seconds` | `1800` | Safety-net exit if no new files appear for this long |
| `archive_retention_days` | `7` | Days to retain archived log files |
| `delete_after_ingest` | `False` | Delete files instead of archiving after ingestion |

---

## 9. YAML Configuration Schema Reference

### Top-Level Structure

```yaml
model: "framework"       # Required. Must be unique per model.
active: true             # Required. Set to false to disable the entire model.
concurrency: 8           # Optional. Max parallel DAG activities (default: 8).

raw:                     # Bronze / ingestion layer
  <objectName>:
    # ... (see Raw Layer properties)

silver:                  # Silver / cleansed layer
  <objectName>:
    # ... (see Silver Layer properties)

gold:                    # Gold / business layer
  <objectName>:
    # ... (see Gold Layer properties)
```

### Raw Layer Properties

```yaml
raw:
  Person:                              # Unique name within this section
    flagActive: 1                      # 1 = active, 0 = skip
    sourceSystemType: sql              # sql | xlsx
    sourceSystemName: sql-weu-dev-...  # Linked service / connection name
    sourceLocationName: adventureworks2022  # Source database name
    sourceObjectName: Person.Person    # Schema.Table in source
    sourceSelectColumns: null          # null = SELECT *, or comma-separated list
    sourceKeyColumns: null             # Key columns (informational)
    destinationSystemName: lh_...      # Destination lakehouse name
    destinationSystemType: lakehouse   # lakehouse | warehouse
    destinationObjectPattern: Person   # Filename prefix in bronze
    destinationDirectoryPattern: adventureworks2022/Person/Person/  # Directory path
    destinationObjectType: parquet     # parquet | csv | avro
    extractType: delta                 # full | delta
    deltaDateColumn: ModifiedDate      # Required when extractType=delta
    deltaDateMinValue: "1900-01-01"    # Optional. Min date for delta (default: 1900-01-01)
    flagBlock: 0                       # 0 = not blocked
    artifactType: pipeline             # pipeline | notebook
    artifactName: pl_copyFrom_adventureWorks2022_kql_yaml
    artifactWorkspace: ""              # Optional. Target workspace name. Empty = current workspace.
    artifactTimeout: 500               # Optional. Timeout in seconds (default: 500)
    artifactParams:                    # Parameters passed to the artifact
      model: model                     # "model" maps to the model parameter value
      sourceObjectName: sourceObjectName  # maps entry attribute to pipeline parameter
    dependsOn: []                      # Optional. List of "<section>.<name>" dependencies
```

### Bronze Layer Additional Properties

These properties exist on `bronze:` entries only (not `raw:`) and are covered in full in [Section 18](#18-landing-zone-and-bronze-change-detection):

```yaml
bronze:
  Customer:
    # ... same required properties as Raw Layer above ...
    requiresLanding: true              # Optional, default false. Route through Landing before bronze.
    piiColumns:                        # Optional. Source columns to hash before promotion to bronze.
      - "CustomerEmail"
      - "CustomerName"
    piiMethod: hash                    # Optional, default hash. encrypt is reserved for future use.
    changeDetection: true              # Optional, default false. Requires extractType: full.
    sourceKeyColumns:                  # Optional even when changeDetection is true - see Section 18.
      - "CustomerId"
```

### Silver Layer Properties

```yaml
silver:
  Person:
    flagActive: 1
    sourceSystemName: lh_lz036_bronze_01
    sourceLocationName: adventureworks2022
    sourceDirectoryPattern: adventureworks2022/Person/Person/
    sourceObjectName: Person.Person
    keyColumns: BusinessEntityID       # Comma-separated key columns for delta merge
    extractType: delta                 # full | delta
    loadType: merge                    # overwrite | merge | append | history — see Section 19 for history
    historicalColumns:                 # Optional. Only used with loadType: history.
      - "Status"                       # Columns tracked as Type-2 (versioned) history.
      - "Segment"                      # Omit entirely to track every column as Type-2.
    destinationObjectPattern: Person/Person_Silver   # Schema/table in silver lakehouse
    destinationDatabase: lh_lz036_silver_01
    artifactType: notebook             # pipeline | notebook
    artifactName: nb_load_silver       # Generic or custom notebook name
    artifactWorkspace: ""              # Optional. Target workspace name.
    artifactTimeout: 500               # Optional. Timeout in seconds.
    artifactParams:
      model: model
      sourceObjectName: objectName
    dependsOn:                         # List of "<section>.<name>" dependencies
      - raw.Person
      - raw.SalesPerson
```

### Gold Layer Properties

```yaml
gold:
  dim_person:
    flagActive: 1
    destinationObjectPattern: framework/dim_person   # Schema/table in gold lakehouse
    destinationDatabase: lh_lz036_gold_01
    artifactType: notebook
    artifactName: nb_dim_person
    artifactWorkspace: ""
    artifactTimeout: 500
    group: dims                        # Optional. Logical group name (e.g., dims, facts)
    artifactParams:
      model: model
    dependsOn:                         # Optional. Explicit object dependencies
      - silver.Person
    dependsOnGroup:                    # Optional. All objects in the named group must complete first
      - dims
```

### Dependency Rules

- `dependsOn` references objects by `<section>.<objectName>` (e.g., `raw.Person`).
- `dependsOnGroup` references logical groups defined by the `group` property (e.g., all `dims` must complete before `facts`).
- Only active objects (`flagActive: 1`) are included in the DAG. Dependencies on inactive objects are automatically filtered out.
- In `nb_yaml_orchestrator_batch`, any `dependsOn` referencing a `raw.*` entry is automatically collapsed into a dependency on `bronze_batch`.

---

## 10. Adding a New Data Source (YAML Mode)

This is the most common development task. Follow these steps to add a new source table end-to-end.

### Step 1 — Add a Raw Entry

Open `ConfigFiles/environments/dev/framework.yml` and add an entry under `raw:`:

```yaml
raw:
  MyNewTable:
    flagActive: 1
    sourceSystemType: sql
    sourceSystemName: my-sql-connection
    sourceLocationName: MyDatabase
    sourceObjectName: dbo.MyTable
    sourceSelectColumns: null
    destinationSystemName: lh_lz036_bronze_01
    destinationSystemType: lakehouse
    destinationObjectPattern: MyNewTable
    destinationDirectoryPattern: MyDatabase/dbo/MyNewTable/
    destinationObjectType: parquet
    extractType: full
    flagBlock: 0
    artifactType: pipeline
    artifactName: pl_copyFrom_adventureWorks2022_kql_yaml   # reuse existing or create new
    artifactParams:
      model: model
      sourceObjectName: sourceObjectName
```

Use `extractType: delta` if you only want to load changed records. In that case also set:
```yaml
    deltaDateColumn: ModifiedDate   # or whatever your change-tracking column is
```

### Step 2 — Add a Silver Entry

```yaml
silver:
  MyNewTable:
    flagActive: 1
    sourceSystemName: lh_lz036_bronze_01
    sourceLocationName: MyDatabase
    sourceDirectoryPattern: MyDatabase/dbo/MyNewTable/
    sourceObjectName: dbo.MyTable
    keyColumns: Id                   # Business key(s) — required for merge/delta loads
    extractType: full
    loadType: overwrite              # overwrite for full loads, merge for delta
    destinationObjectPattern: dbo/MyNewTable_Silver
    destinationDatabase: lh_lz036_silver_01
    artifactType: notebook
    artifactName: nb_load_silver     # generic notebook handles standard loads
    artifactParams:
      model: model
      sourceObjectName: objectName
    dependsOn:
      - raw.MyNewTable
```

> If the table requires custom transformation logic, create a new notebook under `Notebooks/10_Silver/`, set `artifactName` to its name, and follow the [notebook development patterns](#14-notebook-development-patterns).

### Step 3 — Add a Gold Entry (if needed)

```yaml
gold:
  dim_my_entity:
    flagActive: 1
    destinationObjectPattern: framework/dim_my_entity
    destinationDatabase: lh_lz036_gold_01
    group: dims
    artifactType: notebook
    artifactName: nb_dim_my_entity
    artifactParams:
      model: model
    dependsOn:
      - silver.MyNewTable
```

Create `nb_dim_my_entity` in `Notebooks/20_Gold/`.

### Step 4 — Validate

```bash
python ConfigFiles/scripts/validate_config.py ConfigFiles/environments/dev/framework.yml
```

Fix any schema violations before deploying.

### Step 5 — Deploy

Commit the YAML file. The CI/CD pipeline deploys it automatically, or deploy manually by uploading to `Files/config/framework.yml` in the Bronze lakehouse.

### Step 6 — Test

Run `pl_master_YamlOrchestrator` with `model=framework` and `modelReset=True`. Monitor progress in the Eventhouse querysets.

---

## 11. Resume After Failure

The YAML orchestrator tracks each activity execution in the Eventhouse `processExecutionLog` table under a `batchExecution` UUID. When you re-run with `modelReset=False`, the orchestrator:

1. Calls `LastFailedBatchProcesses('<model>')` KQL function to retrieve the last failed batch.
2. Builds a `status_map` of `{processInternalId: status}`.
3. Rebuilds the DAG **only** for activities that had status `Failed` or that never started.
4. Reuses the original `batchExecution` UUID so all retry events share the same batch.
5. Dependency edges are adjusted so that already-succeeded activities are not re-executed.

**To resume after a failure:**

In the pipeline or when triggering the orchestrator notebook, set:
```
modelReset = False
```

> If no prior failed batch exists, the orchestrator falls back to a fresh run regardless of `modelReset`.

---

## 12. Cross-Workspace Execution

You can execute artifacts from a different Fabric workspace by setting `artifactWorkspace` on any YAML entry:

```yaml
silver:
  SharedTransform:
    flagActive: 1
    artifactType: notebook
    artifactName: nb_shared_transform
    artifactWorkspace: shared-analytics-workspace   # Must match the workspace Display Name
    # ... other required properties
```

**Requirements:**
- The identity running the orchestrator must have at least **Contributor** access in the target workspace.
- The target workspace must contain the artifact with the exact `artifactName` display name.
- Cross-workspace notebook execution uses `notebookutils.notebook.run()` with the `workspaceId` parameter. Refer to the [Microsoft documentation](https://learn.microsoft.com/en-us/fabric/data-engineering/notebook-utilities) for known limitations.

When `artifactWorkspace` is empty or omitted, execution falls back to the current workspace.

---

## 13. Variable Library

The `vl_lz036_01` Variable Library (`Variables/vl_lz036_01.VariableLibrary/`) is the single source of truth for all environment-specific IDs.

`CommonFunctions.GetFabricIds()` reads from the Variable Library first and falls back to Fabric API calls. The resolved IDs are cached in Spark session config to avoid repeated API calls.

**Discovery logic** (in `GetVariableLibraryName()`):

1. Check Spark config cache (`fabric.variable_library_name`)
2. Look for a Variable Library matching pattern `^vl_.+_01$`
3. Fall back to any library starting with `vl-` and ending with `-01`
4. Use the first available library if no pattern matches

This makes the framework portable across environments without hardcoding names.

### Updating Variable Library for a New Environment

1. Open `Variables/vl_lz036_01.VariableLibrary/variables.json`.
2. Update all ID values for the target environment.
3. Commit and sync to Fabric, or update directly in the portal.

---

## 14. Notebook Development Patterns

### Standard Notebook Header

Every notebook must start with:

```python
%run CommonFunctions
```

This initialises global variables `WORKSPACE_ID`, `BRONZE_LAKEHOUSE_ID`, `SILVER_LAKEHOUSE_ID`, `GOLD_LAKEHOUSE_ID` and makes all shared functions available.

**Do not** set a default lakehouse in the notebook metadata. Leave it empty to ensure portability across environments and Spark sessions.

### Accepting Orchestrator Parameters

The orchestrator passes parameters to your notebook via `nb_wrapper`. Define them in a **Parameters cell**:

```python
# PARAMETERS CELL
model = "framework"
batchExecution = ""
context = ""
# Add your custom parameters here
```

### Reading Configuration

```python
# Get YAML config for this object
config = GetConfigMetadataFromYaml(model=model, objectName="MyTable", layer="silver")

# config is a Row object with all YAML properties as attributes
source_dir = config.sourceDirectoryPattern
key_cols = config.keyColumns
load_type = config.loadType
```

### Processing Date for Delta Loads

```python
# The orchestrator may pass a preloaded date to avoid KQL queries
if preloadedDate:
    SetPreloadedDate(preloadedDate)

# Get start date for delta filter (uses preloaded date if set, otherwise queries KQL)
start_date = GetStartDate(model, objectName, scope="Silver")
```

### Writing to Silver / Gold

Use the standard merge/overwrite helpers from `CommonFunctions`:

```python
# Full overwrite
WriteToLakehouse(df, SILVER_LAKEHOUSE_ID, "dbo/MyTable_Silver", mode="overwrite")

# Delta merge
MergeToLakehouse(df, SILVER_LAKEHOUSE_ID, "dbo/MyTable_Silver", key_columns=["Id"])
```

### SCD Type 2 Dimensions

```python
# For slowly changing dimensions
MergeHistoricalDim(
    df_new=df,
    target_table_path=f"abfss://{WORKSPACE_ID}@onelake.dfs.fabric.microsoft.com/{GOLD_LAKEHOUSE_ID}/Tables/framework/dim_my_entity",
    key_columns=["BusinessKey"],
    tracked_columns=["Name", "Category"]
)
```

### Logging from a Custom Notebook

If you need to log progress from a notebook that is called directly (not via `nb_wrapper`), use the async logging pattern:

```python
%run nb_custom_logging

# Log activity start
writeLogStarted(
    processName="silver_MyCustomNotebook",
    processInternalId="silver_MyCustomNotebook",
    workspaceId=WORKSPACE_ID,
    context=context
)

# ... your processing logic ...

# Log completion
writeLogFinishedSuccess(
    processName="silver_MyCustomNotebook",
    processInternalId="silver_MyCustomNotebook",
    workspaceId=WORKSPACE_ID,
    context=context
)
```

---

## 15. CI/CD and Deployment

> **Azure DevOps only**: All CI/CD pipeline files in the `CICD/` folder are written for **Azure DevOps Pipelines**. They will not work with GitHub Actions or any other CI/CD platform without being rewritten. If your project uses GitHub, treat the provided files as a reference for the logic and recreate the equivalent workflows as GitHub Actions.

### YAML Config File Deployment

The `deploy-configFiles.yml` Azure DevOps pipeline:
- **Triggers**: On changes to `ConfigFiles/environments/**`
- **Validates**: All YAML files against `ConfigFiles/scripts/config-schema.json`
- **Deploys**: Updated files to the target environment's Bronze lakehouse at `Files/config/`
- **Environments**: Deploys to `dev` on push to `dev` branch, `test` on push to `test` branch, `prod` requires manual approval

> The YAML files deployed by this pipeline are your **project-specific model configurations**, not the example files. Make sure you have created and validated your own `<yourModel>.yml` before enabling the pipeline (see [Section 6.2](#62-create-the-yaml-configuration-file)).

To deploy manually using PowerShell:

```powershell
cd ConfigFiles/scripts
./deploy-configOnelake.ps1 -Environment dev -Model <yourModel>
```

### Fabric Artifact Deployment

Use `deploy-cross-environment.yml` to promote Fabric artifacts (notebooks, pipelines) between environments. This pipeline uses the Fabric deployment pipeline API and is also **Azure DevOps only**.

### Validating YAML Locally

```bash
# Python (recommended)
pip install jsonschema pyyaml
python ConfigFiles/scripts/validate_config.py ConfigFiles/environments/dev/<yourModel>.yml

# PowerShell
ConfigFiles/scripts/Validate-Config.ps1 -ConfigFile ConfigFiles/environments/dev/<yourModel>.yml
```

---

## 16. Monitoring and Troubleshooting

### Eventhouse Querysets

Open `eh_lz036_01_queryset1` in the Fabric portal to run KQL queries against the execution logs.

Useful queries:

```kql
// All executions for the latest batch
processExecutionLog
| where batchExecution == "<uuid>"
| project timestamp, processName, status, errorMessage
| order by timestamp asc

// Failed activities in the last 24 hours
processExecutionLog
| where timestamp > ago(24h) and status == "Failed"
| project timestamp, model, processName, errorMessage
| order by timestamp desc

// Resume eligibility check — last failed batch per model
LastFailedBatchProcesses("framework")

// Processing dates (delta load windows)
lastDatesConfig
| where model == "framework"
| project tableName, scope, startDate
```

### Checking Async Log Files

If the monitor notebook fails to ingest logs, files remain in the log folders. You can inspect them:

1. Navigate to the Bronze lakehouse → Files → logs
2. Check `processExecutionLog/` and `copyDataLog/` for unprocessed JSON files
3. Run `nb_log_file_monitor` manually with appropriate parameters to flush them

### Common Log File Issues

| Symptom | Likely Cause | Resolution |
|---|---|---|
| Monitor exits early | Stale `.done` sentinels from a previous run | Monitor cleans up stale sentinels at startup automatically |
| Log files accumulate but not ingested | KQL connection not initialised | Check that Variable Library has correct `eventhouseName` and `eventhouseDatabase` |
| Monitor exits due to idle timeout | An activity crashed before writing its `.done` sentinel | Check `nb_wrapper` execution logs; increase `idle_timeout_seconds` if needed |
| `Files/logs/archive/` growing large | `archive_retention_days` too high or cleanup not running | Reduce `archive_retention_days` in `nb_log_file_monitor` parameters |

---

## 17. Common Pitfalls

### Default Lakehouse Set in Notebook

**Problem**: If a notebook has a default lakehouse configured in its metadata, it breaks when called from `notebookutils.notebook.runMultiple` in a different Spark session context.  
**Fix**: Always leave the default lakehouse empty in notebook metadata. Access lakehouses via their IDs using the global variables (`BRONZE_LAKEHOUSE_ID` etc.).

### Dependency on Inactive Objects

**Problem**: You add `dependsOn: [raw.MyTable]` but `MyTable` has `flagActive: 0`. The DAG activity is created but the dependency is silently filtered out, causing unexpected execution order.  
**Fix**: Only declare dependencies on objects with `flagActive: 1`. Activate the dependency first if it is needed.

### Hardcoded Workspace or Lakehouse IDs

**Problem**: IDs hardcoded in notebooks or pipelines break deployment to other environments.  
**Fix**: Always use global variables from `CommonFunctions` (`WORKSPACE_ID`, `BRONZE_LAKEHOUSE_ID` etc.) or the Variable Library.

### Configuration Mode Mismatch

**Problem**: Calling `GetConfigMetadata()` (Database mode function) instead of `GetConfigMetadataFromYaml()` when YAML is active.  
**Fix**: Detect the mode with `spark.conf.get("UseYaml", "false").lower() == "true"` and call the appropriate function.

### Silver Activities Starting Before Bronze in Batch Mode

**Problem**: When using `nb_yaml_orchestrator_batch`, you expect a specific silver activity to start as soon as one bronze table is ready, but it has to wait for the entire batch.  
**Fix**: This is the expected behaviour of the batch mode. Use `nb_yaml_orchestrator` (standard) if you need per-table cross-layer dependency.

### Missing `.done` Sentinels Causing Monitor to Hang

**Problem**: `nb_log_file_monitor` keeps running past the expected drain period.  
**Fix**: Check the `nb_wrapper` execution log for errors in `_write_done_sentinel()`. The monitor has an `idle_timeout_seconds` safety net (default: 1800 s) and will exit automatically if no new files appear.

### YAML Validation Errors

**Problem**: Deploy fails because the YAML file does not match the schema.  
**Fix**: Run `validate_config.py` locally before committing. The schema is in `ConfigFiles/scripts/config-schema.json`. Check for missing required properties (e.g., `artifactType`, `artifactName`, `flagActive`).

---

## 18. Landing Zone and Bronze Change Detection

### Why a Landing Zone

Ingestion pipelines write raw extracts into a **Landing lakehouse** before anything is promoted to Bronze. This decouples reprocessing/backfills from the source system (replay from Landing instead of re-querying production), gives PII a bounded window before it's hashed away, and provides a recovery point if the promotion logic itself turns out to be buggy. It is **not** a long-term store — that's still Bronze's job.

Landing uses the exact same dated-folder convention as Bronze (`{yyyy}/{mm}/{dd}/{hh}/{mm}/`), so nothing about how Silver discovers Bronze files changes.

### Architecture — This Is Not a New Orchestrator Layer

There is deliberately **no `landing:` YAML section** and no new DAG layer in the YAML orchestrator. Landing/promotion is dispatched as an **extra pipeline activity** inside the existing ingestion pipelines (`pl_copyFrom_navision`, `pl_lz036_copyFromSource`), gated by the `requiresLanding` property on the same `bronze:` entry that already exists:

```
Copy activity (source → Landing, if requiresLanding: true)
        │
        ▼
nb_promote_landing_to_bronze  (Notebooks/00_Bronze/)
        │  - hashes any configured piiColumns
        │  - if changeDetection: true, diffs against bronze's _current_state
        ▼
Bronze (same dated-folder convention Silver already expects)
```

If `requiresLanding` is `false` or omitted, the Copy activity writes straight to Bronze as before — nothing about this feature changes behavior for a table that doesn't opt in.

### New Bronze Properties

| Property | Required? | Description |
|---|---|---|
| `requiresLanding` | Optional, default `false` | Routes this table's ingestion through Landing and `nb_promote_landing_to_bronze` before Bronze. |
| `piiColumns` | Optional | List of source column names to SHA-256 hash during promotion. Requires `requiresLanding: true`. |
| `piiMethod` | Optional, default `hash` | PII handling method. `encrypt` is reserved for a future reversible option — not implemented yet. |
| `changeDetection` | Optional, default `false` | Diffs the incoming full extract against Bronze's `_current_state` cache and writes only new/changed/deleted rows. **Requires `extractType: full`** — it exists for tables with no reliable watermark column; a table that already supports `extractType: delta` gets change detection for free from the source query and doesn't need this. |
| `sourceKeyColumns` | Optional | Business key column(s) used to identify "the same record" across runs when `changeDetection: true`. **Not required** — if omitted, the promotion notebook falls back to using the entire row as its own key, which still works correctly but means a changed record surfaces as a delete+insert pair instead of an in-place update (there's no identifier independent of content to say it's the same record). Set it explicitly whenever the source table has a real key. |

Example (a full-extract reference table with no natural delta column):

```yaml
bronze:
  DimensionSetEntry:
    # ... required bronze properties ...
    extractType: full
    requiresLanding: true
    changeDetection: true
    sourceKeyColumns:
      - "Dimension Set ID"
      - "Dimension Code"
```

### The `_current_state` Cache

`changeDetection` needs a baseline to diff each run's full extract against. Rather than reconstructing that baseline from Bronze's entire (ever-growing) history on every run, `nb_promote_landing_to_bronze` maintains one small, disposable cache per table — `_current_state`, a sibling folder of the table's dated folders (e.g. `Files/IGENOMIXNAV/dbo/ARGENTINA$DimensionSetEntry/_current_state/`, **not** nested under any one run's dated folder). It holds just `ctr_hash_key`/`ctr_hash_validate` for every row as of the last run, gets fully overwritten every run, and is safe to lose — if it ever disappears, the next run just treats everything as new (one oversized catch-up write, not data loss, since Bronze's actual historical folders are untouched).

Rows the notebook detects as deleted (present in `_current_state`, absent from today's extract) are written as **tombstones** — a row with `ctr_is_bronze_deleted = 1` and business columns nulled — rather than silently dropped. Bronze itself never deletes anything from its historical folders; a deletion is recorded as an additional row, not an erasure.

### Downstream Consequence for Silver

Because `changeDetection` converts a full-extract table into a changed-rows-only Bronze feed, the corresponding **Silver entry must be configured with `extractType: delta`**, even though `bronze.extractType` is `full` for the same table — these are independent fields per layer. Silver's merge logic (and the history load type in [Section 19](#19-silver-history-scd2-load-type)) knows to treat an incoming `ctr_is_bronze_deleted = 1` row as a closure, not a content update.

### Storage Tiers for Landing Retention

Landing's retention/tiering policy (how long raw extracts stay before deletion) is configured as an **OneLake Lifecycle Management rule scoped to the Landing lakehouse item** — it is infrastructure configuration, not a YAML property, and must not be set as the workspace's default storage tier (that would also affect Bronze/Silver/Gold, which need to stay hot). See [OneLake storage tiers](https://learn.microsoft.com/en-us/fabric/onelake/onelake-storage-tiers) — note the preview feature's minimum retention periods (30 days for cool, 90 days for cold) when choosing a retention window, since moving or deleting data before the minimum incurs an early-deletion charge.

---

## 19. Silver History (SCD2) Load Type

### What It Does

`loadType: history` keeps full version history in Silver with explicit `ctr_start_timestamp`/`ctr_end_timestamp`/`ctr_is_active` columns, instead of overwriting the previous version in place (`merge`) or keeping only date-partitioned snapshots (`append`). It's implemented by `HistoryLoad()` in `Standardized_CommonFunctions.Notebook`.

```yaml
silver:
  GL_Entry:
    extractType: delta                 # must be delta if the bronze source uses changeDetection
    loadType: history
    historicalColumns:                 # optional
      - "Status"
```

| loadType | History kept? | Use when |
|---|---|---|
| `overwrite` | No — latest only | Simple lookup/reference tables |
| `merge` | No — SCD1, updates in place | You only need the current state, with a soft-delete flag |
| `append` | Yes — one snapshot per load date | You want simple date-partitioned snapshots, not point-in-time queries |
| `history` | Yes — explicit valid-from/valid-to intervals | You need to answer "what was true as of date X" |

### `historicalColumns`

Optional list of columns tracked as Type-2 (versioned) history. **Omit it entirely** to track every non-key column as Type-2 (the default). Any column *not* listed when `historicalColumns` is set is treated as Type-1 — updated in place on the current row without opening a new version.

### Dates Are Timestamps, Not Day-Granularity Integers

`ctr_start_timestamp`/`ctr_end_timestamp` are full timestamps, not `yyyyMMdd` integers — this matters if the Silver load ever runs more than once in the same calendar day. A day-integer scheme would compute a closing row's end date as "the day before," which can fall *before* that same row's own start date if it was opened earlier the same day. Using real timestamps (and computing the close time as "just before the new version," bumping forward when a collision is detected) keeps every interval valid regardless of run frequency. The very first load backdates all rows to a fixed `1900-01-01` sentinel rather than the load date — the records didn't start being true on the day the pipeline first ran, they simply started being tracked then.

### Deletion Handling

If the upstream Bronze source has `changeDetection: true`, a deleted record arrives in Silver as a row with `ctr_is_bronze_deleted = 1` rather than simply disappearing from the feed. `HistoryLoad()` treats this as a **closure** (`ctr_is_active = 0`, `ctr_end_timestamp` set) — the closed row is kept, not erased. For a Silver table whose `extractType` is `full` instead, a key that's active in Silver but absent from the incoming extract altogether is also treated as a closure, since a full extract is guaranteed to be the complete current universe.

---

*For architecture decisions and setup rationale, see [README.md](../README.md).*  
*For YAML schema details, see [ConfigFiles/README.md](../ConfigFiles/README.md).*
