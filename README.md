# Cosmos DB Live Container Migration for Microsoft Fabric

A set of PySpark notebooks for migrating Cosmos DB containers using Microsoft Fabric. These are a **Fabric-friendly adaptation** of the Databricks-based migration samples from the Azure Cosmos DB Spark Connector repository.  In particular, these are handy moving data across tenants where source and target endpoints are different:

> **Original source:** [DatabricksLiveContainerMigration](https://github.com/Azure/azure-sdk-for-java/tree/main/sdk/cosmos/azure-cosmos-spark_3/Samples/DatabricksLiveContainerMigration)

## Important: Count-Based Completion for Live Sources

These notebooks use a **count-based completion strategy** that works reliably even when the source container is actively receiving writes:

1. The change feed stream runs continuously, copying documents from source to target
2. Periodically (every 2 minutes by default), actual document counts are queried on both source and target containers
3. When `target_count >= source_count` (within a configurable tolerance), the stream stops and migration is complete
4. If the gap stops shrinking (source writes outpace migration), the notebook stops and reports a warning — the operator can then increase throughput, throttle source writes, or run validation to identify the remaining gap

This approach is immune to:
- **Throttling** — slow stream batches don't cause false completion signals
- **Duplicate change feed events** — stream progress counters (`numInputRows`) are used only for display, never for completion decisions
- **Checkpoint resume** — actual counts reflect reality regardless of checkpoint state
- **Active source writes** — stall detection prevents infinite loops; the tolerance parameter handles minor count differences

## Notebooks

### Migration

| Notebook | Role | Description |
|----------|------|-------------|
| `04_CosmosDB_Parallel_Container_Migration` | Orchestrator | Reads a CSV config file listing container pairs, then runs migrations in parallel using `notebookutils.notebook.runMultiple()` |
| `03_CosmosDB_Container_Migration` | Worker (child) | Migrates a single container using Cosmos DB change feed streaming with count-based completion. Called by the orchestrator with injected parameters |

### Validation

| Notebook | Role | Description |
|----------|------|-------------|
| `05_CosmosDB_Parallel_Container_Validation` | Orchestrator | Reads the same CSV config and runs validations in parallel |
| `CosmosDBLiveContainerMigrationValidation` | Worker (child) | Validates a single container migration by comparing source and target document counts and checking for missing documents via left-anti join |

## Execution Flow

```
04_Parallel_Migration (reads CSV, builds DAG)
  └─► 03_Container_Migration  (×N containers, in parallel)

05_Parallel_Validation (reads same CSV, builds DAG)
  └─► CosmosDBLiveContainerMigrationValidation  (×N containers, in parallel)
```

## Setup

### 1. Fabric Environment with Cosmos Spark Connector

Before running these notebooks, you need a Fabric environment with the Azure Cosmos DB Spark Connector JAR. Follow the official guide to set this up:

> **Setup guide:** [Use Spark notebooks with Azure Cosmos DB for NoSQL in Microsoft Fabric](https://learn.microsoft.com/en-us/fabric/database/cosmos-db/how-to-use-spark-notebooks)

Key steps:
- Create a **Fabric Environment** (e.g., `CosmosDB_Migration_Env`)
- Add the `azure-cosmos-spark_3-5_2-12` JAR as a library
- Attach this environment to your notebooks

### 2. Lakehouse

Create a Lakehouse named `Cosmos_Migration` (or update the `%%configure` cells to match your Lakehouse name). This stores:
- The CSV configuration file
- Change feed checkpoints for resumable migrations

### 3. CSV Configuration File

Upload a file named `cosmosDBLiveMigrationList.csv` to `Files/` in the Lakehouse with these columns:

| Column | Description |
|--------|-------------|
| `cosmosSourceEndpoint` | Source Cosmos DB account URI |
| `cosmosSourceMasterKey` | Source account primary key |
| `cosmosRegion` | Preferred region (e.g., `[East US]`) |
| `cosmosSourceDatabaseName` | Source database name |
| `cosmosSourceContainerName` | Source container name |
| `cosmosSourceContainerThroughputControl` | Throughput control % (e.g., `0.95`) |
| `cosmosTargetEndpoint` | Target Cosmos DB account URI |
| `cosmosTargetMasterKey` | Target account primary key |
| `cosmosTargetDatabaseName` | Target database name |
| `cosmosTargetContainerName` | Target container name |
| `cosmosTargetContainerPartitionKey` | Target partition key (e.g., `/pk`) |
| `cosmosTargetContainerProvisionedThroughput` | Autoscale max throughput (e.g., `10000`) |

> **Security note:** This CSV contains credentials. Do not commit it to source control. It is excluded via `.gitignore`.

### 4. Mark Parameter Cells

In the Fabric notebook UI, mark the parameter cells in `03_CosmosDB_Container_Migration` and `CosmosDBLiveContainerMigrationValidation` as **parameter cells** so that `notebookutils.notebook.run()` can inject arguments correctly.

## Key Features

- **Count-based completion** — Periodic queries on actual source/target document counts determine when migration is done (not stream progress counters)
- **Stall detection** — If the document gap stops shrinking (source writes outpace migration), the notebook stops and reports a warning instead of running forever
- **Active source support** — Works reliably with containers that receive constant writes; configurable tolerance handles minor count differences
- **Resumable** — Checkpoints stored in Lakehouse Files; interrupted migrations resume from where they left off
- **Throughput control** — Configurable source throughput throttling to avoid impacting production workloads
- **Automatic retry** — Failed container migrations are retried automatically (configurable)

## Tuning Parameters

The following constants in `03_CosmosDB_Container_Migration` (Step 6) can be adjusted:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `COUNT_CHECK_INTERVAL_SECONDS` | `120` | How often to query source/target counts. Increase for very large containers to reduce overhead |
| `MATCH_TOLERANCE` | `100` | Absolute document-count tolerance. If `source - target <= tolerance`, migration is considered complete. Increase for very active sources |
| `MAX_STALL_CHECKS` | `5` | Stop if the gap hasn't shrunk for this many consecutive count checks. Prevents infinite loops when source writes outpace migration |
| `MAX_TOTAL_SECONDS` | `86400` | Safety timeout (24 hours). Prevents runaway execution |
| `PROGRESS_POLL_SECONDS` | `10` | How often to print stream batch progress (display only, not used for completion) |

## Troubleshooting: Clearing Checkpoints

The migration uses Spark Structured Streaming checkpoints to track progress. This allows interrupted migrations to resume from where they left off. However, a stale or corrupted checkpoint from a previous failed run can cause problems — typically the stream will start but report no data being migrated, even though documents are missing in the target.

To force a **full re-copy from scratch**, set the `clear_checkpoint` parameter to `"true"` in the `03_CosmosDB_Container_Migration` parameter cell (or pass it from the parent notebook's CSV/arguments). This deletes the existing checkpoint so the change feed restarts from the beginning.

Once the migration completes successfully, set it back to `"false"` for any subsequent runs.

> **When to use `clear_checkpoint = "true"`:**
> - The migration shows "Waiting for first data batch..." indefinitely but the target is missing documents
> - A previous run failed partway through and retrying doesn't make progress
> - You want to redo a migration from scratch for any reason

## Key Differences from the Databricks Version

| Feature | Databricks | Fabric |
|---------|-----------|--------|
| Parallelism | Scala `Future` threading | `notebookutils.notebook.runMultiple()` DAG |
| File storage | DBFS | Lakehouse Files |
| JAR management | `%pip install` / cluster library | Fabric Environment |
| Language | Scala | PySpark |
| Checkpoints | DBFS paths | Lakehouse `Files/` paths |
