# Apache Spark Performance Optimization — Learning Series

A hands-on Jupyter notebook series for mastering Apache Spark performance tuning. Each notebook builds on the previous, covering query plan analysis, data skew, caching, partitioning, and bucketing through runnable PySpark examples.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Setup](#setup)
- [Notebooks Overview](#notebooks-overview)
- [Learning Path](#learning-path)
- [Key Concepts Reference](#key-concepts-reference)
- [Data Files](#data-files)

---

## Prerequisites

| Requirement | Version |
|---|---|
| Python | 3.8+ |
| Apache Spark | 3.2+ |
| Java | 8 or 11 |
| JupyterLab / Jupyter Notebook | Any recent version |
| PySpark | Matching your Spark version |

---

## Setup

### 1. Install Python dependencies

```bash
pip install -r requirements.txt
```

### 2. Verify your Spark installation

```bash
spark-submit --version
```

### 3. Prepare input data

The notebooks expect source CSV files in a `data/data_skew/` directory relative to the repo root:

```
data/
└── data_skew/
    ├── raw_customers.csv
    ├── raw_transactions.csv
    ├── Spotify_Listening_Activity.csv
    └── Spotify_Songs.csv
```

Start with **notebook `1_generate_skewed_data.ipynb`** to generate the Parquet datasets used by subsequent notebooks.

### 4. Launch JupyterLab

```bash
jupyter lab
```

Run notebooks in numerical order starting from `1_generate_skewed_data.ipynb`.

---

## Notebooks Overview

### Module 1 — Data Generation

#### `1_generate_skewed_data.ipynb`
Generates the foundational datasets used throughout the series. Loads raw customer and transaction CSVs, intentionally introduces data skew by concentrating ~17.5 million transactions onto a single customer ID (`C0YDPQWPBJ`), and writes the output to Parquet.

**Produces:**
- `data/data_skew/customers.parquet` — 5,000 unique customers
- `data/data_skew/transactions.parquet` — heavily skewed transaction records

**Key APIs:** `SparkSession`, `Window`, `F.row_number()`, UDFs, `.write.parquet()`

---

### Module 2 — Query Plans & Data Skew

#### `2_reading_query_plans.ipynb`
Teaches how to read and interpret Spark query execution plans end-to-end: from parsed logical plan through Catalyst optimization to the physical execution plan.

**Topics:**
- Narrow vs. wide transformations
- Parsed, analyzed, and optimized logical plans
- Physical plan execution strategies
- Predicate pushdown mechanics and its limits with complex types
- Sort-Merge Join vs. Broadcast Join plans

**Key APIs:** `.explain(True)`, `.filter()`, `.join()`, `.repartition()`, `.groupBy().agg()`

**Config used:** `spark.sql.autoBroadcastJoinThreshold = -1`

---

#### `2_skew_dataset_simulation.ipynb`
Demonstrates the real-world performance impact of data skew. Benchmarks a skewed join and shows how a single hot key causes severe partition imbalance.

**Topics:**
- Uniform vs. skewed data distributions
- Measuring join execution time
- Visualizing partition imbalance with `F.spark_partition_id()`

**Key APIs:** `spark.range()`, `.union()`, `F.spark_partition_id()`

**Config used:** `spark.sql.adaptive.enabled = false`

**Benchmark result:** Skewed join on 17.5M-row hot key took ~28 seconds.

---

### Module 3 — DAGs & Solving Skew

#### `3_reading_query_DAGs.ipynb`
Explains Spark's Directed Acyclic Graph (DAG) execution model. Covers how narrow and wide transformations produce different DAG shapes, and how to read the Spark UI's DAG visualization.

**Topics:**
- Narrow transformations: `filter`, `withColumn`, `select`
- Wide transformations: `join`, `groupBy`, `repartition`, `coalesce`
- Sort-Merge Join vs. Broadcast Join DAG shapes
- Using `write.format("noop")` to trigger execution without output

**Key APIs:** `F.broadcast()`, `.write.format("noop").mode("overwrite").save()`

**Config used:** `spark.sql.files.maxPartitionBytes = 256MB`, Kryo serializer

---

#### `3_solving_data_skew_aqe_broadcast.ipynb`
Demonstrates two solutions for data skew: Adaptive Query Execution (AQE) and Broadcast Join.

**Topics:**
- Enabling AQE for automatic skew detection and split
- Broadcast join as an explicit optimization
- Measuring performance improvement for each approach

**Key APIs:** `spark.conf.set("spark.sql.adaptive.enabled", "true")`, `spark.conf.set("spark.sql.adaptive.skewedJoin.enabled", "true")`, `F.broadcast(df)`

| Strategy | Execution Time |
|---|---|
| No optimization | ~11.3 seconds |
| AQE enabled | ~9.6 seconds (~15% faster) |
| Broadcast join | ~2.8 seconds (~75% faster) |

---

### Module 4 — Caching & Salting

#### `4_caching.ipynb`
Covers DataFrame caching strategies and storage level trade-offs.

**Topics:**
- `StorageLevel` options: `MEMORY_ONLY`, `MEMORY_ONLY_SER`, `MEMORY_AND_DISK`, `MEMORY_AND_DISK_DESER`, `DISK_ONLY`
- Serialized vs. deserialized storage trade-offs (CPU vs. memory)
- How cached DataFrames appear as `InMemoryRelation` in physical plans
- When to use `.cache()` vs. `.persist(StorageLevel.XXX)`
- Releasing cache with `.unpersist()`

**Key APIs:** `.cache()`, `.persist(StorageLevel.XXX)`, `.unpersist()`

| Storage Level | Memory Use | CPU Cost | Best For |
|---|---|---|---|
| `MEMORY_ONLY` | High | Low | Repeated reads, abundant memory |
| `MEMORY_ONLY_SER` | Medium | Medium | Memory-constrained environments |
| `MEMORY_AND_DISK` | Variable | Low | Datasets that may exceed memory |
| `MEMORY_AND_DISK_DESER` | Variable | Low | Default; balanced approach |
| `DISK_ONLY` | Low | High | Very large datasets, rare re-reads |

---

#### `4_salting.ipynb`
Teaches the salting technique for manually resolving data skew in joins and aggregations.

**Topics:**
- Artificially introducing skew for demonstration
- Salting joins: add a random salt key, explode the smaller DataFrame
- Salting aggregations: multi-stage groupBy approach
- Verifying even partition distribution after salting

**Key APIs:** `F.rand()`, `F.array()`, `F.explode()`, multi-stage `.groupBy().agg()`

**Config used:** `spark.sql.shuffle.partitions = 3`

**Before/after:** Single partition with 1M rows → three balanced partitions of ~333K rows each.

---

### Module 5 — Partitioning

#### `5_0_partitioning.ipynb`
Comprehensive guide to writing and reading partitioned data for query performance.

**Topics:**
- Single-level and multi-level `partitionBy` during writes
- Partition pruning: Spark skips irrelevant partitions based on filters
- Controlling file count per partition with `repartition()` / `coalesce()`
- Impact of `spark.sql.files.maxPartitionBytes` on read-time parallelism
- Time-series partitioning strategies

**Key APIs:** `.write.partitionBy("col")`, `.write.partitionBy("col1", "col2")`, `.repartition(N)`, `.coalesce(N)`, `F.to_date()`, `F.hour()`

**Config used:** `spark.sql.files.maxPartitionBytes = 256MB`

---

#### `5_1_dynamic_partition_pruning.ipynb`
Shows how Spark's Dynamic Partition Pruning (DPP) optimizes joins involving partitioned datasets.

**Topics:**
- DPP: how Spark evaluates one side of a join to determine which partitions to read from the other side
- Reading `dynamicpruningexpression` in `.explain(True)` output
- `SubqueryAdaptiveBroadcast` plan nodes
- Combining DPP with Broadcast-Hash Joins

**Key APIs:** `.write.partitionBy("listen_date")`, complex join conditions, `.explain(True)`

---

### Module 6 — Bucketing

#### `6_0_bucketing.ipynb`
Explains bucketing as an alternative to partitioning for optimizing joins and aggregations.

**Topics:**
- Writing bucketed tables with `.bucketBy(num_buckets, col=...)`
- How bucketing eliminates shuffle in joins (no `Exchange` operator)
- Bucket pruning when filtering on the bucketed column
- Bucketing vs. partitioning: when to use each
- Reading bucketed tables with `spark.table()`

**Key APIs:** `.write.bucketBy(4, col="product_id").saveAsTable("name")`, `spark.table("name")`, `F.col()`

**Config used:** `spark.sql.autoBroadcastJoinThreshold = -1`

| Join Type | Plan includes `Exchange`? | Notes |
|---|---|---|
| Non-bucketed | Yes (shuffle) | Expensive for large tables |
| Bucketed (same buckets) | No | Shuffle eliminated |
| Bucketed + filter | No | Bucket pruning reduces I/O |

---

## Learning Path

```
1_generate_skewed_data          ← Start here: build the datasets
        │
        ├── 2_reading_query_plans        ← Understand query plans
        ├── 2_skew_dataset_simulation    ← See the skew problem
        │
        ├── 3_reading_query_DAGs         ← Understand DAG execution
        ├── 3_solving_data_skew_aqe_broadcast  ← Fix skew with AQE & broadcast
        │
        ├── 4_caching                    ← Control data persistence
        ├── 4_salting                    ← Fix skew manually
        │
        ├── 5_0_partitioning             ← Organize data for pruning
        ├── 5_1_dynamic_partition_pruning ← Advanced join optimization
        │
        └── 6_0_bucketing                ← Eliminate join shuffles
```

---

## Key Concepts Reference

| Concept | Notebook | Summary |
|---|---|---|
| Data Skew | `2_skew_dataset_simulation` | One partition gets disproportionately more data, slowing the entire job |
| Predicate Pushdown | `2_reading_query_plans` | Catalyst pushes filters into the data scan to reduce rows read |
| AQE | `3_solving_data_skew_aqe_broadcast` | Spark dynamically adjusts the plan at runtime based on actual data stats |
| Broadcast Join | `3_solving_data_skew_aqe_broadcast` | Small table broadcast to all executors, eliminating shuffle |
| Salting | `4_salting` | Add a random key suffix to distribute hot keys across partitions |
| Storage Levels | `4_caching` | Controls where cached data lives (memory, disk, serialized) |
| Partition Pruning | `5_0_partitioning` | Spark reads only the relevant directory partitions matching query filters |
| Dynamic Partition Pruning | `5_1_dynamic_partition_pruning` | Spark computes relevant partitions at runtime from the other join side |
| Bucketing | `6_0_bucketing` | Pre-sort data into buckets so joins on the bucketed column skip shuffle |
| Sort-Merge Join | `2_reading_query_plans` | Default join strategy; both sides sorted then merged — requires shuffle |

---

## Data Files

| File | Used In | Notes |
|---|---|---|
| `raw_customers.csv` | Notebook 1 | Source customer data |
| `raw_transactions.csv` | Notebook 1 | Source transaction data |
| `customers.parquet` | Notebooks 2–4 | Generated by notebook 1; 5K customers |
| `transactions.parquet` | Notebooks 2–4 | Generated by notebook 1; skewed ~17.5M rows on one ID |
| `Spotify_Listening_Activity.csv` | Notebooks 5, 6 | Source listening data for partitioning demos |
| `Spotify_Songs.csv` | Notebook 5.1 | Song metadata with release dates |
| `orders.csv` | Notebook 6 | 1,000 order records for bucketing demo |
| `products.csv` | Notebook 6 | 100 product records for bucketing demo |
