# Cosmos DB Live Container Migration for Microsoft Fabric

A set of PySpark notebooks for migrating Cosmos DB containers using Microsoft Fabric. These are a **Fabric-friendly adaptation** of the Databricks-based migration samples from the Azure Cosmos DB Spark Connector repository:

> **Original source:** [DatabricksLiveContainerMigration](https://github.com/Azure/azure-sdk-for-java/tree/main/sdk/cosmos/azure-cosmos-spark_3/Samples/DatabricksLiveContainerMigration)

## Important: Snapshot Copy, Not Continuous Sync

These notebooks are designed to **copy data from a source container to a target container** using a point-in-time snapshot approach. They count the source documents at the start of the migration and stop once that many rows have been transferred.

**This is not intended for continuous synchronization.** If your source container is constantly being updated, these notebooks will copy all documents that exist at migration start but will not keep the target in sync with ongoing changes. For continuous replication scenarios, consider [Azure Cosmos DB Change Feed](https://learn.microsoft.com/en-us/azure/cosmos-db/change-feed) or other purpose-built replication solutions.

## Notebooks

### Migration

| Notebook | Role | Description |
|----------|------|-------------|
| `04_CosmosDB_Parallel_Container_Migration` | Orchestrator | Reads a CSV config file listing container pairs, then runs migrations in parallel using `notebookutils.notebook.runMultiple()` |
| `03_CosmosDB_Container_Migration` | Worker (child) | Migrates a single container using Cosmos DB change feed streaming. Called by the orchestrator with injected parameters |

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

- **Parallel execution** — Configurable concurrency (default: 4 containers in parallel)
- **Resumable** — Checkpoints stored in Lakehouse Files; interrupted migrations resume from where they left off
- **Checkpoint-aware completion** — Counts existing target documents to correctly determine remaining work on resumed runs
- **Throughput control** — Configurable source throughput throttling to avoid impacting production workloads
- **Automatic retry** — Failed container migrations are retried automatically (configurable)

## Key Differences from the Databricks Version

| Feature | Databricks | Fabric |
|---------|-----------|--------|
| Parallelism | Scala `Future` threading | `notebookutils.notebook.runMultiple()` DAG |
| File storage | DBFS | Lakehouse Files |
| JAR management | `%pip install` / cluster library | Fabric Environment |
| Language | Scala | PySpark |
| Checkpoints | DBFS paths | Lakehouse `Files/` paths |
