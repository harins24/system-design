# MapReduce and Batch Processing

**Phase 6 — Distributed Systems + Architecture | Topic 11**

---

## What is Batch Processing?

Batch processing means processing large amounts of data collected over a period of time, all at once, rather than processing each record as it arrives.

```
Stream processing (real-time):
  Process each event as it arrives
  Kafka consumer reads OrderCreated → updates counter immediately
  Low latency: milliseconds

Batch processing:
  Collect data over hours/days
  Process all at once in a scheduled job
  Higher latency: minutes to hours

  "Run the revenue report for last month"
  "Compute recommendations for all users weekly"
  "Calculate payroll for all employees end of month"
  "Retrain ML model on last 30 days of data"
```

---

## The Problem Batch Processing Solves

```
Naive approach — run queries on production DB:
  "Revenue by product category for last year"
  SELECT category, SUM(total)
  FROM orders o
  JOIN order_items oi ON oi.order_id = o.id
  JOIN products p ON p.id = oi.product_id
  WHERE o.created_at > '2025-01-01'
  GROUP BY category

  On 1 billion rows:
  → Query runs for 4 hours
  → Production DB CPU at 100%
  → All other queries slow/fail
  → Users see degraded service

Batch processing solution:
  Don't run on production DB
  Export data to dedicated processing cluster
  Process in parallel across many machines
  Return results without affecting production
```

---

## MapReduce — The Foundation

Google's 2004 paper introduced MapReduce: a programming model for processing large datasets across a cluster.

### The Core Idea

```
Any large computation can be expressed as two phases:

MAP:    Apply a function to each input record
        Produces intermediate key-value pairs
        Embarrassingly parallel — each record independent

REDUCE: Group by key, apply aggregate function
        One reducer per unique key
        Parallelized by key

Between: SHUFFLE — sort and group map output by key
         Route all same-key pairs to same reducer
```

### MapReduce — Word Count Example

```
Input:
  File 1: "the cat sat on the mat"
  File 2: "the cat ate the rat"

MAP phase (each word → (word, 1)):
  Mapper 1 (File 1):
    (the, 1), (cat, 1), (sat, 1), (on, 1), (the, 1), (mat, 1)

  Mapper 2 (File 2):
    (the, 1), (cat, 1), (ate, 1), (the, 1), (rat, 1)

SHUFFLE phase (group by key):
  (ate,  [1])
  (cat,  [1, 1])
  (mat,  [1])
  (on,   [1])
  (rat,  [1])
  (sat,  [1])
  (the,  [1, 1, 1, 1])

REDUCE phase (sum values per key):
  (ate,  1)
  (cat,  2)
  (mat,  1)
  (on,   1)
  (rat,  1)
  (sat,  1)
  (the,  4)
```

### MapReduce — Revenue by Category

```
Input: 1 billion order records

MAP (extract category → revenue):
  {orderId: 1, category: "Electronics", total: 99.99}
  → (Electronics, 99.99)

  {orderId: 2, category: "Books", total: 24.99}
  → (Books, 24.99)

  {orderId: 3, category: "Electronics", total: 149.99}
  → (Electronics, 149.99)

  1000 mappers process 1M records each (parallel)

SHUFFLE:
  (Books,       [24.99, 12.99, ...])
  (Electronics, [99.99, 149.99, ...])
  (Clothing,    [49.99, 79.99, ...])

REDUCE (sum per category):
  (Books,       $4,829,374.23)
  (Electronics, $12,847,293.11)
  (Clothing,    $3,291,847.55)

  100 reducers process different categories in parallel
  Result computed in minutes across 1100 machines
  Production DB untouched
```

---

## Apache Hadoop — MapReduce Implementation

```
Hadoop architecture:

  HDFS (Hadoop Distributed File System):
    Store large files split across many machines
    Each file split into 128MB blocks
    Each block replicated 3x for fault tolerance

  YARN (resource manager):
    Allocates CPU and memory across cluster

  MapReduce runtime:
    Schedules map and reduce tasks on worker nodes
    Moves computation to data (not data to computation)
    Handles task failures (re-runs failed tasks)

"Move computation to data":
  Data (1TB) on 100 worker nodes
  Instead of: load 1TB to one machine, run computation
  Hadoop does: send map code to each node, run locally
  Only results (small) transferred over network
  → Much less network IO → faster
```

### Hadoop MapReduce in Java

```java
// Classic MapReduce — still runs in production at large companies

// Mapper
public class RevenueMapper
    extends Mapper<LongWritable, Text, Text, DoubleWritable> {

    private final Text category = new Text();
    private final DoubleWritable revenue = new DoubleWritable();

    @Override
    public void map(LongWritable key, Text value, Context context)
            throws IOException, InterruptedException {

        // Parse CSV line: orderId,category,total
        String[] fields = value.toString().split(",");
        String cat = fields[1];
        double total = Double.parseDouble(fields[2]);

        category.set(cat);
        revenue.set(total);
        context.write(category, revenue);
    }
}

// Reducer
public class RevenueReducer
    extends Reducer<Text, DoubleWritable, Text, DoubleWritable> {

    @Override
    public void reduce(Text key, Iterable<DoubleWritable> values,
                       Context context)
            throws IOException, InterruptedException {

        double sum = 0;
        for (DoubleWritable val : values) {
            sum += val.get();
        }

        context.write(key, new DoubleWritable(sum));
    }
}

// Job configuration
Job job = Job.getInstance(conf, "Revenue by Category");
job.setMapperClass(RevenueMapper.class);
job.setReducerClass(RevenueReducer.class);
job.setNumReduceTasks(100);  // 100 parallel reducers
FileInputFormat.addInputPath(job, new Path("/data/orders/2025/"));
FileOutputFormat.setOutputPath(job, new Path("/results/revenue/"));
job.waitForCompletion(true);
```

---

## Apache Spark — Modern Batch Processing

Hadoop MapReduce has major limitations:
- Writes intermediate results to HDFS (slow disk I/O)
- Only two-phase (map + reduce) — complex jobs need many MapReduce jobs chained
- Hard to write iterative algorithms (ML, graph processing)

Spark solves all of these:

```
Spark vs Hadoop MapReduce:

Processing:    In-memory (RDDs/DataFrames) vs disk-based
Speed:         100x faster for iterative algorithms
API:           Rich functional API vs low-level map/reduce
Languages:     Scala, Java, Python, R vs mainly Java
SQL:           Spark SQL (native) vs Hive on top of Hadoop
Streaming:     Spark Streaming / Structured Streaming vs nothing
ML:            MLlib built-in vs external tools
```

### Spark Core Concepts

```
RDD (Resilient Distributed Dataset):
  Immutable, distributed collection of objects
  Partitioned across cluster
  Lazy evaluation: transformations not executed until action
  Fault tolerant: can recompute from lineage on failure

DataFrame:
  RDD of structured data (like a distributed SQL table)
  Schema-aware: column names and types
  Highly optimized via Catalyst query optimizer

Dataset:
  Type-safe DataFrame (Java/Scala)
  Compile-time type checking

Transformation (lazy):
  Returns new DataFrame, doesn't execute
  map, filter, groupBy, join, select, where

Action (triggers execution):
  Starts actual computation
  count, collect, show, write, save
```

### Spark — Revenue by Category

```java
// Spark SQL (most common approach)
SparkSession spark = SparkSession.builder()
    .appName("Revenue by Category")
    .config("spark.sql.shuffle.partitions", "200")
    .getOrCreate();

// Read from data lake (S3, HDFS, or Delta Lake)
Dataset<Row> orders = spark.read()
    .parquet("s3://data-lake/orders/year=2025/");

// SQL-like transformations
Dataset<Row> revenue = orders
    .filter(col("created_at").gt(lit("2025-01-01")))
    .join(
        spark.read().parquet("s3://data-lake/products/"),
        orders.col("product_id").equalTo(col("id"))
    )
    .groupBy("category")
    .agg(
        sum("total").alias("total_revenue"),
        count("order_id").alias("order_count"),
        avg("total").alias("avg_order_value")
    )
    .orderBy(desc("total_revenue"));

// Write results to output
revenue.write()
    .mode(SaveMode.Overwrite)
    .parquet("s3://results/revenue-by-category/2025/");

// Or use Spark SQL directly
orders.createOrReplaceTempView("orders");

Dataset<Row> result = spark.sql("""
    SELECT
        p.category,
        SUM(o.total) as total_revenue,
        COUNT(o.id) as order_count
    FROM orders o
    JOIN products p ON o.product_id = p.id
    WHERE o.created_at > '2025-01-01'
    GROUP BY p.category
    ORDER BY total_revenue DESC
    """);
```

### Spark Execution Model

```
Query: groupBy("category").agg(sum("total"))

Spark creates execution plan:
  Stage 1 (Map): Read parquet files, apply filter, partial aggregation
    Task 1: Process partition 1 (local sum per category)
    Task 2: Process partition 2 (local sum per category)
    Task N: Process partition N (local sum per category)

  SHUFFLE: Redistribute by category (all rows for same category → same node)

  Stage 2 (Reduce): Final aggregation per category
    Task 1: Sum Electronics results from all Stage 1 tasks
    Task 2: Sum Books results from all Stage 1 tasks

Data stored in memory between stages (not written to disk like Hadoop)
→ Much faster for multi-stage jobs
```

---

## Data Lake Architecture

Modern batch processing organizes data in a data lake:

```
Raw Zone (Bronze):
  All data exactly as received
  Immutable, never modified
  Append-only
  Retained forever (compliance)
  Format: JSON, CSV, Avro (raw)

  Source: Kafka → S3/GCS/ADLS raw zone
  Batch job runs every hour, dumps Kafka to S3

Cleaned Zone (Silver):
  Deduplicated, validated, type-correct
  Joins across related datasets
  Format: Parquet (columnar, compressed)
  Partitioned by date for efficient queries

  Spark job: Bronze → validate → dedupe → Silver

Aggregated Zone (Gold):
  Pre-computed aggregates for reporting
  Business metrics, KPIs
  Format: Parquet or Delta Lake
  Ready for BI tools (Tableau, Looker)

  Spark job: Silver → aggregate → Gold

Medallion architecture:
  Each zone serves different consumers
  Raw: data scientists, compliance
  Cleaned: ML model training, data engineering
  Aggregated: business users, BI dashboards
```

---

## Delta Lake — Reliability for Data Lakes

Raw data lakes (plain S3) have problems:

```
Problems with plain S3 data lake:
  No ACID transactions:
    Spark job writes 50 files to S3
    Job fails midway: 25 files written, 25 not
    Data lake in inconsistent state

  No schema enforcement:
    New pipeline writes order.total as string instead of decimal
    Downstream jobs break silently

  No time travel:
    Overwrite a partition → previous data gone forever
    "What did the data look like last week?"

Delta Lake solves all:
  ACID transactions on top of S3
  Schema enforcement and evolution
  Time travel (version history)
  Efficient upserts (MERGE operations)
```

```python
# Delta Lake with PySpark

from delta.tables import DeltaTable

# Write with ACID guarantee
df.write \
    .format("delta") \
    .partitionBy("date") \
    .save("s3://data-lake/silver/orders")

# Upsert (MERGE) — handle late-arriving data
delta_table = DeltaTable.forPath(spark, "s3://data-lake/silver/orders")

delta_table.alias("existing").merge(
    new_data.alias("updates"),
    "existing.order_id = updates.order_id"
).whenMatchedUpdateAll() \
 .whenNotMatchedInsertAll() \
 .execute()

# Time travel — query yesterday's data
df = spark.read \
    .format("delta") \
    .option("versionAsOf", 5) \
    .load("s3://data-lake/silver/orders")

# Or by timestamp
df = spark.read \
    .format("delta") \
    .option("timestampAsOf", "2026-01-15") \
    .load("s3://data-lake/silver/orders")
```

---

## Batch Processing Patterns

### Partitioning for Efficiency

```
Unpartitioned data lake:
  s3://data-lake/orders/  (100TB of files, mixed dates)

  Query: "Orders in January 2026"
  → Spark must scan all 100TB to find January records
  → Slow, expensive

Partitioned by date:
  s3://data-lake/orders/year=2026/month=01/  (small subset)

  Query: "Orders in January 2026"
  → Spark reads only /year=2026/month=01/ partition
  → Partition pruning: scan 1/24 of data = 24x faster

Partition key selection:
  Choose by most common filter in queries
  Date is almost always a good partition key
  Don't over-partition: too many small files = slow

  Ideal partition size: 128MB - 1GB
  Too small: too many files, overhead
  Too large: can't parallelize well
```

### Incremental Processing

```
Full reprocess (simple but expensive):
  Every run: read ALL data, recompute everything
  Run time grows with total data size
  Eventually too slow

Incremental processing (efficient):
  Only process new/changed data since last run

  Pattern:
  Last processed timestamp: "2026-02-13 00:00:00"

  New run:
  SELECT * FROM source WHERE updated_at > '2026-02-13 00:00:00'
  Process → merge into target

  Save new watermark: "2026-02-14 00:00:00"

  Spark Structured Streaming handles this automatically:
  Checkpoint file tracks last processed Kafka offset
  Each micro-batch processes only new messages
```

### Broadcast Joins — Small Table Optimization

```
Join: 1 billion orders × 10,000 products

Standard join: shuffle both datasets by product_id
  → Move 1 billion order rows across network
  → Very expensive

Broadcast join: copy small table to every node
  products (10,000 rows = few MB) → replicate to all workers
  Each worker joins its orders partition locally
  → No order rows moved over network
  → Much faster
```

```java
// Spark broadcasts automatically for small tables
// Or hint explicitly:
Dataset<Row> result = orders.join(
    broadcast(products),  // force broadcast
    orders.col("product_id").equalTo(products.col("id"))
);
```

---

## Batch vs Stream Processing Decision

```
Use Batch Processing when:
  ✅ Latency of hours/days acceptable
  ✅ Complex analytics on historical data
  ✅ ML model training
  ✅ End-of-period reports (monthly revenue, payroll)
  ✅ Data warehouse ETL
  ✅ Large-scale data transformation

Use Stream Processing when:
  ✅ Real-time decisions needed (fraud detection)
  ✅ Live dashboards (current active users)
  ✅ Event-driven workflows (order placed → notify immediately)
  ✅ Continuous metric aggregation

Kappa Architecture (stream handles both):
  Batch = stream with infinite retention + replay
  One system (Kafka + Flink) handles both
  Replay Kafka from beginning = batch reprocessing
  Real-time consumption = stream processing
```

---

## Key Takeaways

```
Batch processing: process large accumulated data in scheduled runs
vs Stream: real-time per-event; vs Batch: scheduled bulk

MapReduce:
  Map: transform each record → key-value pairs
  Shuffle: group all same-key pairs together
  Reduce: aggregate values per key
  Parallelized at map and reduce phase
  Foundation of all distributed data processing

Hadoop:
  HDFS: distributed file system (128MB blocks, 3x replicated)
  YARN: resource management
  MapReduce: disk-based execution (slow but fault tolerant)

Spark:
  In-memory processing (100x faster than Hadoop for iterative)
  RDD/DataFrame API
  Rich transformations + Spark SQL
  Lazy evaluation: plan first, execute on action
  Used for: ETL, ML training, data lake processing, reporting

Data Lake Medallion Architecture:
  Bronze (raw): exactly as received, immutable
  Silver (clean): deduplicated, validated, typed
  Gold (aggregate): pre-computed metrics for BI

Delta Lake:
  ACID transactions on S3
  Schema enforcement
  Time travel (query historical versions)
  Upserts via MERGE

Optimization patterns:
  Partitioning: partition pruning → scan fraction of data
  Broadcast join: replicate small table → avoid shuffle
  Incremental: process only new/changed data

Batch in your context:
  Order analytics, revenue reports → Spark on S3
  ML feature engineering → Delta Lake
  Daily aggregations for dashboards → Gold layer
```
