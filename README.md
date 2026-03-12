# ADF-ETL-PIPELINE

# 🔄 ADF ETL Pipeline

An end-to-end **Extract, Transform, Load (ETL)** pipeline built with **Azure Data Factory (ADF)**, enabling automated data ingestion, transformation, and loading across cloud data sources and sinks.

---

## 📌 Overview

This project demonstrates a production-style ETL pipeline using Azure Data Factory. It covers the full pipeline lifecycle — from raw data ingestion through linked services, transformation via ADF data flows, to loading into a target data store — all defined as code and version-controlled.

---

## 🏗️ Architecture

```
Data Source(s)
     │
     ▼
[Linked Service]  ←── Connection configurations
     │
     ▼
[Integration Runtime]  ←── Execution environment (Azure / Self-Hosted)
     │
     ▼
[Dataset]  ←── Schema & format definitions
     │
     ▼
[Pipeline]  ←── Orchestration & transformation logic
     │
     ▼
Target Data Store (Azure SQL / Blob / ADLS / etc.)
```

---

## 📁 Repository Structure

```
ADF-ETL-PIPELINE/
├── ADF/                    # Core ADF resource definitions
├── dataset/                # Dataset definitions (source & sink schemas)
├── factory/                # ADF factory-level configuration
├── integrationRuntime/     # Integration runtime configurations
├── linkedService/          # Linked service connection definitions
├── pipeline/               # Pipeline JSON definitions (ETL orchestration)
├── publish_config.json     # ADF publish configuration
└── README.md
```

### Folder Details

| Folder | Description |
|---|---|
| `ADF/` | Top-level ADF resource configuration files |
| `dataset/` | Defines the structure and format of source and destination datasets |
| `factory/` | Factory-level settings and metadata |
| `integrationRuntime/` | Configuration for Azure or Self-Hosted Integration Runtimes |
| `linkedService/` | Connection strings and credentials for data sources/sinks |
| `pipeline/` | Core ETL pipeline definitions with activities and control flow |

---

## ⚙️ Prerequisites

- **Azure Subscription** with access to Azure Data Factory
- **Azure Data Factory** instance (v2)
- **Azure Storage Account** or other target data store
- **Git Integration** enabled on your ADF instance (for CI/CD)
- ADF author access (`Data Factory Contributor` role or higher)

---

## 🚀 Getting Started

### 1. Clone the Repository

```bash
git clone https://github.com/yogesh5121/ADF-ETL-PIPELINE.git
cd ADF-ETL-PIPELINE
```

### 2. Connect ADF to This Repository

1. Open your **Azure Data Factory** instance in the Azure Portal.
2. Navigate to **Manage → Git Configuration**.
3. Select **GitHub** as the repository type.
4. Connect to this repo and set the **collaboration branch** to `main`.
5. Set the **publish branch** to `adf_publish`.

### 3. Configure Linked Services

1. In ADF Studio, go to **Manage → Linked Services**.
2. Update the connection details (storage account keys, SQL credentials, etc.) for each linked service defined in the `linkedService/` folder.
3. Test each connection to confirm connectivity.

### 4. Run the Pipeline

1. Navigate to **Author → Pipelines** in ADF Studio.
2. Select the desired pipeline.
3. Click **Debug** to run with a test payload, or **Add Trigger → Trigger Now** for a full run.
4. Monitor run status under **Monitor → Pipeline Runs**.

---

## 🔗 Key Components

### Linked Services
Defines connections to external data sources and sinks (e.g., Azure Blob Storage, Azure SQL Database, REST APIs). Credentials are managed via **Azure Key Vault** integration.

### Datasets
Describes the schema, format (CSV, JSON, Parquet, etc.), and location of data used by pipeline activities.

### Integration Runtime
The compute infrastructure used to execute data movement and transformation:
- **Azure IR** — fully managed, used for cloud-to-cloud transfers
- **Self-Hosted IR** — used for on-premises or private network data sources

### Pipelines
Contains the ETL orchestration logic, including:
- **Copy Activity** — moves data from source to sink
- **Data Flow Activity** — applies transformations (filter, join, aggregate, etc.)
- **Control Flow** — conditional execution, loops, and error handling

---

## 📊 Pipeline Scenarios

| # | Pipeline Name | Source | Sink | Pattern |
|---|---|---|---|---|
| 1 | `PL_BLOB_TO_ADLS` | Azure Blob Storage (CSV) | ADLS Gen2 (Parquet) | Simple copy + format conversion |
| 2 | `PL_CSV_TO_ADLS` | ADLS Gen2 (CSV wildcard) | ADLS Gen2 (Parquet) | Multi-file wildcard ingestion |
| 3 | `PL_CSV_TO_ADLS_DEL_SOURCE` | Azure Blob Storage (CSV) | ADLS Gen2 (Parquet) | Copy + delete source after load |
| 4 | `PL_MS_SQL_TO_ADLS` | SQL Server (single table) | ADLS Gen2 (Parquet) | Metadata validation + copy |
| 5 | `PL_MULTIPLE_TABLE_SQL_TO_ADLS` | SQL Server (all dbo tables) | ADLS Gen2 (Parquet) | Dynamic multi-table loop |
| 6 | `PL_REST_API_TO_ADLS` | REST API (JSON) | ADLS Gen2 (Parquet) | API ingestion + field mapping |

---

### 1. `PL_BLOB_TO_ADLS` — Blob Storage to ADLS (CSV → Parquet)

**Purpose:** Ingests raw CSV files from Azure Blob Storage and writes them to ADLS Gen2 in Parquet format for downstream analytics.

```
Azure Blob Storage (CSV)
         │  [DS_BLOB_TO_ADLS]
         ▼
  ┌─────────────────────┐
  │   CPY_BLOB_ADLS     │
  │   Copy Activity     │
  │  • Recursive read   │
  │  • Type conversion  │
  └─────────────────────┘
         │  [DS_PQ1_BLOB_TO_ADLS]
         ▼
ADLS Gen2 (Parquet)
```

| Property | Value |
|---|---|
| Activity | `CPY_BLOB_ADLS` — Copy |
| Source | Azure Blob Storage — CSV (`DS_BLOB_TO_ADLS`) |
| Sink | ADLS Gen2 — Parquet (`DS_PQ1_BLOB_TO_ADLS`) |
| Recursive Read | ✅ Yes |
| Staging | ❌ No |
| Timeout | 12 hrs · Retry: 0 |
| Type Conversion | ✅ Enabled · Truncation allowed · Boolean ≠ Number |

---

### 2. `PL_CSV_TO_ADLS` — Wildcard Multi-CSV to ADLS (CSV → Parquet)

**Purpose:** Scans an ADLS Gen2 folder using a `*.csv` wildcard to pick up **all CSV files at once** and consolidates them into a single Parquet output. Ideal for batch landing zones where multiple files arrive before processing.

```
ADLS Gen2 / Landing Zone
  (all *.csv files)
         │  [DS_MUTLIPLE_CSV]
         │  Wildcard: *.csv
         │  Recursive: true
         ▼
  ┌──────────────────────┐
  │   CPY_MULTIPLE_CSV   │
  │   Copy Activity      │
  │  • Wildcard pattern  │
  │  • All CSVs merged   │
  │  • Type conversion   │
  └──────────────────────┘
         │  [DS_CSV_TO_PARQET]
         ▼
ADLS Gen2 (Parquet)
```

| Property | Value |
|---|---|
| Activity | `CPY_MULTIPLE_CSV` — Copy |
| Source | ADLS Gen2 — CSV wildcard `*.csv` (`DS_MUTLIPLE_CSV`) |
| Sink | ADLS Gen2 — Parquet (`DS_CSV_TO_PARQET`) |
| Wildcard Pattern | `*.csv` (all CSV files in folder) |
| Recursive Read | ✅ Yes |
| Staging | ❌ No |
| Timeout | 12 hrs · Retry: 0 |
| Type Conversion | ✅ Enabled · Truncation allowed · Boolean ≠ Number |

**Key Difference from Pipeline 1:** Source store is **ADLS Gen2** (not Blob Storage) and uses a wildcard filename filter to batch-process multiple CSV files in a single activity run.

---

### 3. `PL_CSV_TO_ADLS_DEL_SOURCE` — Copy CSV to ADLS then Delete Source

**Purpose:** Implements a **move** pattern — copies CSV data from Blob Storage to ADLS Gen2 as Parquet, then **deletes the source file** only after a successful copy. Ensures no data loss while keeping the source clean and avoiding duplicate processing.

```
Azure Blob Storage (CSV)
         │  [DS_BLOB_CSV]
         ▼
  ┌──────────────────┐
  │   CPY_DATA_CSV   │   ── on Success ──►  ┌──────────────────┐
  │   Copy Activity  │                       │  DEL_CSV_SOURCE  │
  │  • CSV → Parquet │                       │  Delete Activity │
  │  • Recursive     │                       │  • Deletes source│
  └──────────────────┘                       │  • [DS_BLOB_DEL] │
         │                                   └──────────────────┘
         │  [DS_PARQUET_ADLS]
         ▼
ADLS Gen2 (Parquet)
```

**Activities:**

| # | Activity | Type | Depends On | Description |
|---|---|---|---|---|
| 1 | `CPY_DATA_CSV` | Copy | — | Copies CSV from Blob → ADLS Parquet |
| 2 | `DEL_CSV_SOURCE` | Delete | `CPY_DATA_CSV` (Succeeded) | Deletes source CSV from Blob Storage |

| Property | Value |
|---|---|
| Source | Azure Blob Storage — CSV (`DS_BLOB_CSV`) |
| Sink | ADLS Gen2 — Parquet (`DS_PARQUET_ADLS`) |
| Delete Dataset | `DS_BLOB_DEL` (Blob source reference) |
| Delete triggered | Only on copy **Success** (safe delete) |
| Logging on Delete | ❌ Disabled |
| Timeout | 12 hrs · Retry: 0 |

**Why this pattern?** Prevents reprocessing of already-ingested files. The conditional dependency (`Succeeded`) ensures source files are never deleted if the copy fails.

---

### 4. `PL_MS_SQL_TO_ADLS` — SQL Server to ADLS with Metadata Validation

**Purpose:** Extracts data from a **single SQL Server table** to ADLS Gen2 as Parquet. Before copying, a **GetMetadata** activity validates that the source table's structure and column count are accessible — acting as a data quality gate before the copy proceeds.

```
SQL Server Table
         │  [DS_SQLSEVER]
         ▼
  ┌──────────────────────┐
  │   Get Metadata1      │   Validates: structure, columnCount
  │   GetMetadata        │
  └──────────────────────┘
         │  on Success
         ▼
  ┌──────────────────────┐
  │   CPY_SQL_ADLS       │
  │   Copy Activity      │
  │  • SQL Server source │
  │  • Query timeout 2hr │
  │  • No partitioning   │
  └──────────────────────┘
         │  [DS_SINK_SQL_TO_ADLS]
         ▼
ADLS Gen2 (Parquet)
```

**Activities:**

| # | Activity | Type | Depends On | Description |
|---|---|---|---|---|
| 1 | `Get Metadata1` | GetMetadata | — | Reads `structure` + `columnCount` from source table |
| 2 | `CPY_SQL_ADLS` | Copy | `Get Metadata1` (Succeeded) | Extracts full table to ADLS as Parquet |

| Property | Value |
|---|---|
| Source | SQL Server (`DS_SOURCE_CPY_SQL`) |
| Sink | ADLS Gen2 — Parquet (`DS_SINK_SQL_TO_ADLS`) |
| Metadata Fields | `structure`, `columnCount` |
| Query Timeout | 2 hours |
| Partition Option | None |
| Staging | ❌ No |
| Timeout | 12 hrs · Retry: 0 |

**Why GetMetadata first?** Catches schema issues or connectivity problems before the copy runs, preventing partial loads and making failures easier to diagnose.

---

### 5. `PL_MULTIPLE_TABLE_SQL_TO_ADLS` — Dynamic Multi-Table SQL to ADLS

**Purpose:** Dynamically discovers **all tables in the `dbo` schema** of a SQL Server database using a Lookup query, then iterates over each table with a ForEach loop, copying every table to ADLS Gen2 as individual Parquet files. No hardcoding of table names required.

```
SQL Server (INFORMATION_SCHEMA)
         │
         ▼
  ┌──────────────────────────────────────┐
  │   Tablelookup                        │
  │   Lookup Activity                    │
  │   SELECT TABLE_NAME                  │
  │   FROM INFORMATION_SCHEMA.TABLES     │
  │   WHERE TABLE_SCHEMA = 'dbo'         │
  │   → returns list of all table names  │
  └──────────────────────────────────────┘
         │  @activity('Tablelookup').output.value
         │  on Success
         ▼
  ┌──────────────────────────────────────┐
  │   ForEach1  (sequential)             │
  │   ForEach Activity                   │
  │   items: all table names             │
  │                                      │
  │   └── ┌──────────────────────────┐  │
  │        │   CPY_MULTI_DATA        │  │
  │        │   Copy Activity         │  │
  │        │  • TableName: @item()   │  │
  │        │  • FlattenHierarchy     │  │
  │        └──────────────────────────┘  │
  └──────────────────────────────────────┘
         │  [DS_PQ_MUTLI_ADLS]
         ▼
ADLS Gen2 (one Parquet file per table)
```

**Activities:**

| # | Activity | Type | Depends On | Description |
|---|---|---|---|---|
| 1 | `Tablelookup` | Lookup | — | Queries `INFORMATION_SCHEMA.TABLES` to get all `dbo` table names |
| 2 | `ForEach1` | ForEach | `Tablelookup` (Succeeded) | Loops over each table name sequentially |
| 2a | `CPY_MULTI_DATA` | Copy (inside ForEach) | — | Copies each table to ADLS as Parquet |

| Property | Value |
|---|---|
| Lookup Query | `SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA = 'dbo'` |
| ForEach Mode | Sequential (`isSequential: true`) |
| Dynamic Parameter | `@item().Table_Name` passed to source dataset |
| Source Dataset | `DS_SRC_MLTI_TO_ADLS` (parameterised by `TableName`) |
| Sink Dataset | `DS_PQ_MUTLI_ADLS` |
| Copy Behaviour | `FlattenHierarchy` — all files written flat to sink folder |
| Timeout | 12 hrs · Retry: 0 per activity |

**Why this pattern?** Fully dynamic — adding or removing tables in SQL Server automatically reflects in the next pipeline run without any ADF changes. Perfect for initial bulk migrations or nightly full-refresh loads.

---

### 6. `PL_REST_API_TO_ADLS` — REST API to ADLS (JSON → Parquet)

**Purpose:** Ingests product pricing data from a **REST API** (GET request with RFC5988 pagination support) and lands it in ADLS Gen2 as Parquet. Applies explicit **field-level JSON path mapping** to flatten and rename API response fields into clean column names.

```
REST API Endpoint
  (HTTP GET · JSON response)
         │  [DS_REST_API_TO_ADLS]
         │  Pagination: RFC5988
         ▼
  ┌──────────────────────────────┐
  │   CPY_DATA_CSV_PQT           │
  │   Copy Activity              │
  │  • RestSource (GET)          │
  │  • JSON path field mapping   │
  │  • Renames spaces → _        │
  │  • 19 fields mapped          │
  └──────────────────────────────┘
         │  [DS_SINK_REST_API_TO_ADLS]
         ▼
ADLS Gen2 (Parquet)
```

| Property | Value |
|---|---|
| Activity | `CPY_DATA_CSV_PQT` — Copy |
| Source | REST API — HTTP GET (`DS_REST_API_TO_ADLS`) |
| Sink | ADLS Gen2 — Parquet (`DS_SINK_REST_API_TO_ADLS`) |
| Request Method | GET |
| HTTP Timeout | 1 min 40 sec |
| Request Interval | 10 ms |
| Pagination | RFC5988 (`Link` header-based) |
| Staging | ❌ No |
| Timeout | 12 hrs · Retry: 0 |

**Field Mappings (JSON Path → Parquet Column):**

| Source JSON Path | Sink Column Name |
|---|---|
| `$['id']` | `id` |
| `$['Sku']` | `Sku` |
| `$['TP 1']` | `TP_1` |
| `$['TP 2']` | `TP_2` |
| `$['index']` | `index` |
| `$['Weight']` | `Weight` |
| `$['Catalog']` | `Catalog` |
| `$['MRP Old']` | `MRP_Old` |
| `$['Ajio MRP']` | `Ajio_MRP` |
| `$['Category']` | `Category` |
| `$['Style Id']` | `Style_Id` |
| `$['Paytm MRP']` | `Paytm_MRP` |
| `$['Amazon MRP']` | `Amazon_MRP` |
| `$['Myntra MRP']` | `Myntra_MRP` |
| `$['Flipkart MRP']` | `Flipkart_MRP` |
| `$['Limeroad MRP']` | `Limeroad_MRP` |
| `$['Snapdeal MRP']` | `Snapdeal_MRP` |
| `$['Final MRP Old']` | `Final_MRP_Old` |
| `$['Amazon FBA MRP']` | `Amazon_FBA_MRP` |

**Domain Context:** The API returns e-commerce product pricing data with MRP values across multiple platforms (Amazon, Flipkart, Myntra, Ajio, Paytm, Snapdeal, Limeroad). Field names with spaces are sanitised to use underscores for Parquet column compatibility.

---

## 🔒 Security

- Sensitive credentials (connection strings, passwords) are **never stored in plain text**.
- All secrets are referenced via **Azure Key Vault** linked service.
- Role-based access control (RBAC) is applied at the ADF and storage levels.

---

## 🛠️ CI/CD

This repository supports ADF's native **Git-based CI/CD**:

- Feature development happens on feature branches.
- Changes are merged to `main` via pull requests.
- Publishing to ADF generates ARM templates on the `adf_publish` branch.
- ARM templates can be deployed to staging/production environments via **Azure DevOps Pipelines** or **GitHub Actions**.

---

## 📄 License

This project is open source. Feel free to fork, modify, and use it as a reference for your own ADF ETL implementations.

---

## 🙋 Author

**Yogesh** — [GitHub Profile](https://github.com/yogesh5121)
