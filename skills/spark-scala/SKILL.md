---
name: spark-scala
description: Expert guidance for Apache Spark development in Scala. Use this skill when the user asks to write Spark jobs, pipelines, transformations, queries, or tests in Scala; when dealing with DataFrames, Datasets, RDDs, Structured Streaming, Spark SQL, joins, aggregations, partitioning, caching, UDFs, or performance tuning; when designing data pipelines or applying TDD to Spark code; or when the user mentions SparkSession, SparkContext, DataFrame, Dataset, parquet, Delta, broadcast, shuffle, watermark, trigger, or any Spark API in Scala. Always use this skill for Spark + Scala work, even if the user only says "write a Spark job" or "process this data with Spark".
---

# Apache Spark with Scala

## Core Philosophy

**Design principles:**
- **Pure transformations**: Spark transformations are pure functions — same input → same output, no side effects. Each `.transform()` call is an algebraic equation; gluing them together is algebraic substitution
- **Immutability**: DataFrames/Datasets are immutable. Every transformation returns a new one
- **Single Responsibility**: one named function per transformation step — composable, readable, testable
- **Dependency Rule**: business logic lives in plain Scala; Spark is the framework at the outer layer. Pure core functions are tested without Spark; Spark transformations wrap them
- **TDD first**: write the test before the transformation. Untestable code signals a hidden side effect

---

## Execution Model

```
Driver (your main program)
  └── SparkContext / SparkSession
        └── DAG Scheduler → breaks plan into Stages
              └── Task Scheduler → sends Tasks to Executors
                    └── Executors (JVM processes on worker nodes)
                          └── Tasks (one per partition)
```

**Narrow vs Wide transformations:**
- **Narrow** (no shuffle): `map`, `filter`, `withColumn`, `select` — each partition produces output from its own data only. Fast.
- **Wide** (shuffle across network): `groupBy`, `join`, `repartition`, `distinct` — data moves between executors. Expensive. Each wide transformation creates a new Stage boundary.

**Lazy evaluation:** Transformations build a logical plan. Nothing executes until an **action** is called (`count`, `collect`, `write`, `show`). Catalyst optimizes the full plan before execution.

```scala
// This builds a plan — zero computation happens
val plan = spark.read.parquet("s3://bucket/orders/")
  .filter($"status" === "active")
  .groupBy($"region")
  .agg(sum($"amount"))

// This triggers the optimized execution
plan.write.parquet("s3://bucket/output/")
```

**Rule: minimize wide transformations. Filter before joining. Cache only when reusing.**

---

## Temp Views — Querying DataFrames with SQL

```scala
// Session-scoped temp view — only visible to this SparkSession
// Automatically dropped when the session ends
df.createOrReplaceTempView("orders")
spark.sql("SELECT region, SUM(amount) AS total FROM orders GROUP BY region")

// Global temp view — visible to all SparkSessions in the same application
// Lives until the SparkContext is stopped; must be queried with the global_temp prefix
df.createOrReplaceGlobalTempView("shared_orders")
spark.sql("SELECT * FROM global_temp.shared_orders")
spark.newSession().sql("SELECT * FROM global_temp.shared_orders")  // different session, same data

// createOrReplaceTempView     → dies with its SparkSession (dev/test friendly)
// createOrReplaceGlobalTempView → survives SparkSessions, dies with SparkContext (cross-job sharing)
```

---

## map() vs flatMap() in Spark

```scala
// map(): one-to-one transformation — always same number of rows as input
df.map(row => row.getString(0).toUpperCase)   // 100 rows in → 100 rows out

// flatMap(): one-to-many — each input row can produce 0, 1, or N output rows
df.flatMap(row => row.getString(0).split(","))  // "a,b,c" → 3 rows: "a", "b", "c"

// Prefer explode() over flatMap for nested arrays in the DataFrame API
import org.apache.spark.sql.functions.explode
df.select(explode($"tags").as("tag"), $"order_id")   // one row per tag

// On typed Datasets, flatMap works directly on domain types
case class Order(id: Long, items: List[Item])
case class Item(sku: String, qty: Int)

val orders: Dataset[Order] = ???
val allItems: Dataset[Item] = orders.flatMap(_.items)  // Dataset[Order] → Dataset[Item]

// map()     — transforms, preserves cardinality, same number of elements
// flatMap() — transforms AND flattens, can change cardinality (0..N per input)
```

---

## SparkSession — The Entry Point

```scala
import org.apache.spark.sql.SparkSession

val spark = SparkSession.builder()
  .appName("OrderPipeline")
  .config("spark.sql.shuffle.partitions", "200")        // default 200, tune per job
  .config("spark.sql.adaptive.enabled", "true")         // AQE: on by default Spark 3.2+
  .config("spark.sql.adaptive.coalescePartitions.enabled", "true")
  .getOrCreate()

import spark.implicits._  // enables .toDS(), .toDF(), $"col" syntax
```

For tests, use a local session:
```scala
val spark = SparkSession.builder()
  .master("local[2]")
  .appName("test")
  .config("spark.sql.shuffle.partitions", "2")  // small for tests
  .getOrCreate()
```

---

## DataFrames vs Datasets — Choose Deliberately

| | `DataFrame` (= `Dataset[Row]`) | `Dataset[T]` |
|---|---|---|
| Type safety | Runtime only | Compile-time (case class) |
| Performance | Same Catalyst optimizer | Same — Encoders avoid Java serialization |
| Best for | SQL-style ETL, schema-driven work | Domain logic, unit testing, type-safe pipelines |
| Available in | Scala, Python, Java, R | Scala and Java only |

**Encoders**: Spark uses `Encoder[T]` to serialize case classes to its internal binary format — faster than Java serialization, no GC overhead. Available automatically via `import spark.implicits._` for case classes.

**Prefer Datasets for domain logic:**
```scala
case class Order(id: Long, customerId: Long, amount: Double, status: String, region: String)

val orders: Dataset[Order] = spark.read
  .parquet("s3://bucket/orders/")
  .as[Order]  // type-safe from here — compiler catches field name typos

// Pure function — testable without Spark
def isActive(o: Order): Boolean = o.status == "active" && o.amount > 0

val active: Dataset[Order] = orders.filter(isActive)
```

**Use DataFrames for SQL-style ETL:**
```scala
import org.apache.spark.sql.functions._

spark.read.parquet("s3://bucket/sales/")
  .filter($"year" === 2024)
  .groupBy($"region", $"category")
  .agg(sum($"amount").as("total"), countDistinct($"customer_id").as("unique_customers"))
  .orderBy(desc("total"))
```

---

## Schema Definition — Always Explicit in Production

Never use schema inference (`inferSchema`) in production — it's a full data scan and often wrong for edge cases.

```scala
import org.apache.spark.sql.types._

val orderSchema = StructType(Array(
  StructField("id",          LongType,      nullable = false),
  StructField("customer_id", LongType,      nullable = false),
  StructField("amount",      DoubleType,    nullable = true),
  StructField("status",      StringType,    nullable = true),
  StructField("created_at",  TimestampType, nullable = true),
  StructField("metadata",    MapType(StringType, StringType), nullable = true)
))

// Or derive schema from case class (preferred with Datasets)
import org.apache.spark.sql.Encoders
val schema = Encoders.product[Order].schema
```

---

## Null Handling

Handle nulls explicitly — never assume a column is non-null.

```scala
import org.apache.spark.sql.functions._

// coalesce: first non-null value
df.withColumn("amount", coalesce($"amount", lit(0.0)))

// fill nulls by type or column
df.na.fill(0.0, Seq("amount", "tax"))
df.na.fill("unknown", Seq("status", "region"))

// drop rows with any null in specified columns
df.na.drop(Seq("id", "customer_id"))

// replace specific values
df.na.replace("status", Map("" -> "unknown", "N/A" -> "unknown"))

// null-safe equality (handles null == null → true)
df.filter($"region" <=> "EU")

// filter out nulls
df.filter($"amount".isNotNull && $"status".isNotNull)
```

---

## Column API — Complete Reference

```scala
import org.apache.spark.sql.functions._

// Column reference forms (all equivalent)
df.select($"price")           // using implicits
df.select(col("price"))       // explicit import
df.select(column("price"))    // alias for col
df.selectExpr("price * 1.21") // SQL expression string
df.select(lit(42))            // literal value

// Column methods
$"name".contains("foo")
$"name".startsWith("ord")
$"name".endsWith("_v2")
$"status".isin("active", "pending")       // IN (...) filter
$"amount".between(10.0, 500.0)            // inclusive range
$"amount".isNull
$"amount".isNotNull
$"tags".getItem(0)                        // array index access
$"metadata".getField("source")            // struct field access
$"price".as("unit_price")                 // alias
$"price".alias("unit_price")              // same as as()
$"price".cast(DoubleType)                 // type cast

// Multiple column renaming / dropping
df.withColumnRenamed("old_name", "new_name")
df.drop("col_a", "col_b", "col_c")        // drop multiple columns at once
```

---

## Aggregations and Window Functions

```scala
import org.apache.spark.sql.functions._
import org.apache.spark.sql.expressions.Window

// Complete aggregation function reference
df.groupBy($"region", $"category")
  .agg(
    count("*").as("total_count"),
    countDistinct($"customer_id").as("unique_customers"),
    sum($"amount").as("total"),
    avg($"amount").as("avg"),
    min($"amount").as("min_order"),
    max($"amount").as("max_order"),
    first($"status").as("first_status"),        // first value in group (non-deterministic)
    last($"status").as("last_status"),           // last value in group
    stddev($"amount").as("stddev"),              // sample standard deviation
    stddev_pop($"amount").as("stddev_pop"),      // population standard deviation
    var_samp($"amount").as("variance"),          // sample variance
    var_pop($"amount").as("variance_pop"),       // population variance
    collect_list($"order_id").as("order_ids"),   // all values including duplicates
    collect_set($"region").as("unique_regions"), // distinct values
    approx_count_distinct($"customer_id", rsd = 0.05).as("approx_customers"), // HLL sketch
    skewness($"amount").as("skew"),
    kurtosis($"amount").as("kurt"),
    corr($"price", $"quantity").as("price_qty_corr"),
    covar_pop($"price", $"quantity").as("covariance"),
    percentile_approx($"amount", 0.95, 100).as("p95")
  )

// Rollup — hierarchical subtotals (year,month), (year,null), (null,null)
df.rollup($"year", $"month")
  .agg(sum($"amount").as("total"))
  .na.fill(Map("year" -> "ALL", "month" -> "ALL"))

// Cube — all 2^N combinations of the grouping columns
df.cube($"year", $"month", $"region")
  .agg(sum($"amount").as("total"))
// Identify totals: .where($"month".isNull) — year subtotals
//                  .where($"year".isNull && $"month".isNull && $"region".isNull) — grand total

// Pivot — one column value per aggregated column
df.groupBy($"region")
  .pivot("month", Seq("Jan", "Feb", "Mar"))  // explicit values avoids full scan
  .agg(sum($"amount"))
// result columns: region | Jan | Feb | Mar

// Window functions — operate within partitions, no groupBy collapse
val byRegion    = Window.partitionBy($"region")
val byRegionOrd = Window.partitionBy($"region").orderBy(desc($"amount"))
val running     = Window.partitionBy($"region").orderBy($"created_at")
                        .rowsBetween(Window.unboundedPreceding, Window.currentRow)

df.withColumn("rank",          rank().over(byRegionOrd))        // 1,2,2,4 (gaps)
  .withColumn("dense_rank",    dense_rank().over(byRegionOrd))  // 1,2,2,3 (no gaps)
  .withColumn("row_number",    row_number().over(byRegionOrd))  // 1,2,3,4 (unique)
  .withColumn("pct_of_region", $"amount" / sum($"amount").over(byRegion))
  .withColumn("running_total", sum($"amount").over(running))
  .withColumn("prev_amount",   lag($"amount", 1).over(running))  // time-series
  .withColumn("next_amount",   lead($"amount", 1).over(running))
```

---

## Joins — Strategy Matters

```scala
// Broadcast join (large + small < spark.sql.autoBroadcastJoinThreshold, default 10MB)
import org.apache.spark.sql.functions.broadcast
val result = orders.join(broadcast(regionLookup), "region_id")

// Sort-merge join (default for large+large, produces a shuffle)
val result = orders.join(customers, orders("customer_id") === customers("id"), "inner")

// SQL hints when you want to override the optimizer
import spark.implicits._
spark.sql("""
  SELECT /*+ BROADCAST(r) */ o.*, r.name as region_name
  FROM orders o JOIN regions r ON o.region_id = r.id
""")

// API hint
orders.hint("broadcast").join(regions, "region_id")

// Join types
orders.join(customers, Seq("customer_id"), "inner")       // only matching rows
orders.join(customers, Seq("customer_id"), "left")        // all orders, null if no customer
orders.join(customers, Seq("customer_id"), "left_semi")   // orders that HAVE a customer (no customer cols)
orders.join(customers, Seq("customer_id"), "left_anti")   // orders with NO customer
```

**Join rules:**
1. Always broadcast the smaller side — if < 10MB it's automatic; force it explicitly otherwise
2. Filter before joining — push predicates down
3. Select only needed columns before joining — reduces shuffle data size
4. AQE handles skew automatically in Spark 3+ — enable `spark.sql.adaptive.skewJoin.enabled`

---

## Partitioning and Performance

```scala
// Repartition (full shuffle) — increase parallelism or balance by a column
val df = rawDf.repartition(200, $"region")

// Coalesce (no shuffle) — reduce partitions before writing
val compact = df.coalesce(8)

// Check partition count
df.rdd.getNumPartitions

// Partition sizes diagnostic
df.rdd.mapPartitions(it => Iterator(it.size)).collect()
// Target: 100–200 MB per partition. Too small = task overhead. Too large = OOM.

// Cache when reusing the same Dataset in multiple actions
val enriched = ordersWithRegions.cache()
enriched.count()  // materialize the cache
val revenue  = enriched.agg(sum("amount")).collect()
val topTen   = enriched.orderBy(desc("amount")).limit(10).collect()
enriched.unpersist()  // free memory when done
```

**Write with partition pruning in mind:**
```scala
// Partition by columns you'll filter on in future reads
df.write
  .mode("overwrite")
  .partitionBy("year", "month", "region")  // future: filter $"year" === 2024 → reads only that dir
  .parquet("s3://bucket/orders/")
```

---

## Idempotent Writes and Deduplication

```scala
// Delta Lake: replaceWhere (idempotent overwrite of a partition)
df.write
  .format("delta")
  .mode("overwrite")
  .option("replaceWhere", "year = 2024 AND month = 1")
  .save("s3://bucket/delta/orders/")

// Delta Lake: MERGE (upsert — insert new, update existing)
import io.delta.tables._

DeltaTable.forPath("s3://bucket/delta/orders/")
  .as("target")
  .merge(newData.as("source"), "target.id = source.id")
  .whenMatched().updateAll()
  .whenNotMatched().insertAll()
  .execute()

// Deduplication within a batch
df.dropDuplicates("id", "event_time")

// Deduplication with row_number (keep latest by timestamp)
import org.apache.spark.sql.expressions.Window
val deduped = df
  .withColumn("rn", row_number().over(Window.partitionBy("id").orderBy(desc("updated_at"))))
  .filter($"rn" === 1)
  .drop("rn")
```

---

## Clean Transformation Design

```scala
// Bad: anonymous lambdas, untestable, unreadable
val result = df
  .filter(r => r.getAs[String]("status") == "active" && r.getAs[Double]("amount") > 0)
  .groupBy("region").agg(sum("amount"))

// Good: named transformations, each testable in isolation
def filterActive(ds: Dataset[Order]): Dataset[Order] =
  ds.filter(o => o.status == "active" && o.amount > 0)

def applyVAT(rate: Double = 0.21)(ds: Dataset[Order]): Dataset[Order] =
  ds.map(o => o.copy(amount = o.amount * (1 + rate)))

def aggregateByRegion(ds: Dataset[Order]): DataFrame =
  ds.groupBy($"region").agg(sum($"amount").as("total"), count("*").as("orders"))

// Pipeline reads top-to-bottom like prose
val report = rawOrders
  .transform(filterActive)
  .transform(applyVAT(rate = 0.21))
  .transform(aggregateByRegion)
```

---

## SQL Hints

```scala
// Broadcast hint in SQL
spark.sql("SELECT /*+ BROADCAST(r) */ * FROM orders o JOIN regions r ON o.region_id = r.id")

// Repartition hint
spark.sql("SELECT /*+ REPARTITION(100) */ * FROM orders")
spark.sql("SELECT /*+ REPARTITION(100, region) */ * FROM orders")

// Coalesce hint (reduce without shuffle)
spark.sql("SELECT /*+ COALESCE(10) */ * FROM orders")

// Force sort-merge join
spark.sql("SELECT /*+ MERGE(o, c) */ * FROM orders o JOIN customers c ON o.customer_id = c.id")
```

---

## UDFs — Use Sparingly

UDFs bypass Catalyst. Use built-in `functions._` first; write a UDF only when no built-in exists.

```scala
// Bad: UDF for something built-in
val myUpper = udf((s: String) => s.toUpperCase)   // just use upper($"col")

// Good: UDF for custom business logic unavailable as built-in
val parseProductCode = udf((raw: String) => Option(raw)
  .filter(_.nonEmpty)
  .map(_.split("-").take(2).mkString("_").toLowerCase)
  .getOrElse("unknown"))

spark.udf.register("parse_product_code", parseProductCode)
val df = orders.withColumn("product_code", parseProductCode($"raw_code"))
```

---

## TDD for Spark — Three Levels

Business logic must be testable without Spark. If a function requires a SparkSession to test, it is mixing concerns.

```scala
// Level 1: pure business logic — NO Spark needed, fast
class OrderLogicSpec extends AnyFunSpec with Matchers {
  describe("isActive") {
    it("returns true for active orders with positive amount") {
      isActive(Order(1L, 10L, 100.0, "active", "EU")) shouldBe true
    }
    it("returns false for zero-amount orders") {
      isActive(Order(2L, 10L, 0.0, "active", "EU")) shouldBe false
    }
    it("returns false for cancelled orders") {
      isActive(Order(3L, 10L, 50.0, "cancelled", "EU")) shouldBe false
    }
  }

  describe("applyVAT") {
    it("increases amount by the VAT rate") {
      applyVAT(0.21)(Order(1L, 10L, 100.0, "active", "EU")).amount shouldBe 121.0
    }
  }
}

// Level 2: Spark transformation — local session, small data
class OrderTransformationsSpec extends AnyFunSpec with Matchers with BeforeAndAfterAll {
  lazy val spark = SparkSession.builder().master("local[2]").appName("test")
    .config("spark.sql.shuffle.partitions", "2").getOrCreate()
  import spark.implicits._

  override def afterAll(): Unit = spark.stop()

  describe("filterActive (Dataset transformation)") {
    it("keeps only active non-zero orders") {
      val input = Seq(
        Order(1L, 10L, 100.0, "active",    "EU"),
        Order(2L, 11L, 0.0,   "active",    "EU"),  // zero amount
        Order(3L, 12L, 50.0,  "cancelled", "EU")   // cancelled
      ).toDS()

      val result = filterActive(input).collect()

      result should have length 1
      result.head.id shouldBe 1L
    }
  }
}

// Level 3: integration — reads real files, writes to temp dir
class OrderPipelineIntegrationSpec extends AnyFunSpec with Matchers {
  it("processes parquet files end-to-end") {
    val tmpDir = Files.createTempDirectory("spark-test").toString
    // write test data → run pipeline → assert output
  }
}
```

**Property-based tests for transformations:**
```scala
import org.scalacheck.Prop.forAll
import org.scalacheck.Gen

val orderGen = for {
  id     <- Gen.posNum[Long]
  amount <- Gen.choose(-100.0, 10000.0)
  status <- Gen.oneOf("active", "cancelled", "pending")
} yield Order(id, 1L, amount, status, "EU")

property("VAT never decreases amount for positive rates") {
  forAll(orderGen, Gen.choose(0.0, 0.5)) { (order, rate) =>
    applyVAT(rate)(order).amount >= order.amount
  }
}
```

**Assert execution plan in tests:**
```scala
// Verify a broadcast join was chosen (no shuffle)
val plan = result.queryExecution.executedPlan.toString
plan should include("BroadcastHashJoin")
plan should not include "SortMergeJoin"
```

---

## spark-submit — Complete Reference

```bash
spark-submit \
  --class com.example.OrderPipeline \
  --master yarn \                          # yarn | spark://host:7077 | k8s://https://host:443 | local[N]
  --deploy-mode cluster \                  # cluster | client (see Deploy Modes below)
  --driver-memory 4g \                     # RAM for the driver JVM
  --driver-cores 2 \                       # cores for the driver (yarn/standalone only)
  --executor-memory 8g \                   # RAM per executor
  --executor-cores 4 \                     # cores per executor
  --num-executors 20 \                     # fixed executor count (disables dynamic allocation)
  --jars lib/extra.jar,lib/util.jar \      # additional JARs added to executor + driver classpath
  --files config/app.properties \          # files distributed to each executor working dir
  --properties-file /etc/spark/job.conf \  # load additional --conf settings from file
  --conf spark.sql.shuffle.partitions=400 \
  --conf spark.sql.adaptive.enabled=true \
  --conf spark.dynamicAllocation.enabled=true \
  --conf spark.dynamicAllocation.minExecutors=5 \
  --conf spark.dynamicAllocation.maxExecutors=50 \
  --conf spark.serializer=org.apache.spark.serializer.KryoSerializer \
  --packages io.delta:delta-core_2.12:2.4.0 \   # Maven coords — downloaded at runtime
  myapp_2.12-1.0.jar \
  --date 2024-01-15                        # application arguments (after the JAR)
```

### Deploy Modes

| Mode | Driver runs on | Use for |
|------|---------------|---------|
| `client` | The machine running spark-submit | Development, interactive, notebooks |
| `cluster` | A worker node inside the cluster | Production — driver HA, low latency to executors |

```bash
# client: you see logs in your terminal; if you disconnect, driver dies
spark-submit --deploy-mode client ...

# cluster: driver restarts on failure; logs are in cluster log aggregation
spark-submit --deploy-mode cluster ...
```

### Cluster Managers

```bash
--master local[4]                     # local development: 4 threads
--master local[*]                     # local: all available cores
--master yarn                         # Hadoop YARN (most common in enterprise)
--master spark://master-host:7077     # Spark Standalone cluster
--master k8s://https://k8s-api:443   # Kubernetes
```

### Config Priority (highest to lowest)

1. `--conf` flag on spark-submit command line
2. `spark.conf.set(...)` called in application code
3. `--properties-file` specified on spark-submit
4. `$SPARK_HOME/conf/spark-defaults.conf`
5. Environment variables in `spark-env.sh`

---

## RDDs — When to Use

Use RDDs only when the DataFrame/Dataset API doesn't support what you need:
- Custom serialization or binary data
- Low-level control (custom partitioners)
- Porting legacy Spark 1.x code

```scala
val rdd = spark.sparkContext.textFile("s3://bucket/logs/*.log")
val errorCounts = rdd
  .filter(_.contains("ERROR"))
  .map(line => (line.split(" ")(2), 1))  // (error_type, 1)
  .reduceByKey(_ + _)
  .sortBy(_._2, ascending = false)

// Convert back to DataFrame when possible
errorCounts.toDF("error_type", "count").show()
```

---

## Reference Files

- `references/data-sources.md` — Parquet, Delta Lake (MERGE/time-travel), JSON, CSV, JDBC, Kafka, Avro with all production options + UDFs + TDD patterns
- `references/optimization.md` — AQE, Catalyst phases, Tungsten, Spark UI interpretation, OOM diagnosis, partition sizing, CBO, skew diagnosis, anti-patterns
- `references/streaming-patterns.md` — Watermarks, trigger modes, type-safe streaming, stateful ops, stream-stream joins, event-time vs processing-time, progress metrics
- `references/error-handling.md` — Dead letter pattern, idempotent writes, corrupt data handling, checkpoint recovery
- `references/rdd-api.md` — Complete RDD API: transformations, pair RDD ops (combineByKey), accumulators (custom AccumulatorV2), broadcast variables, lineage checkpointing
- `references/spark-ml.md` — MLlib Pipeline API: all feature transformers, all estimators, CrossValidator, ParamGridBuilder, 4-level TDD, common pitfalls
