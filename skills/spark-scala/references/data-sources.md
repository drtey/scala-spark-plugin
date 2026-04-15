# Spark Data Sources Reference

## Format Selection Guide

| Format | Use when | Avoid when |
|--------|----------|------------|
| **Parquet** | General analytics, columnar queries, wide tables | Write-heavy workloads needing ACID |
| **Delta Lake** | ACID, upserts, time travel, streaming+batch unified | No Delta support in environment |
| **ORC** | Hive ecosystem, high compression | Non-Hive environments |
| **Avro** | Schema evolution, Kafka integration, row-oriented | Analytical/columnar query patterns |
| **JSON** | Semi-structured, schema flexibility | High-volume analytics (slow parse) |
| **CSV** | Legacy interop, human-readable exchange | Production analytics |
| **JDBC** | Relational DB integration | Large table full scans |

---

## Parquet

```scala
// Read
val df = spark.read
  .option("mergeSchema", "true")      // schema evolution across files
  .option("pathGlobFilter", "*.parquet")
  .parquet("s3://bucket/orders/year=2024/")

// Read with partition pruning (columns from directory names are auto-inferred)
val df = spark.read.parquet("s3://bucket/orders/")
  .filter($"year" === 2024 && $"month" === 1)  // pushes filter to file scan

// Write with partitioning
df.write
  .mode("overwrite")                        // overwrite | append | ignore | errorIfExists
  .option("compression", "snappy")          // snappy (default) | gzip | lz4 | zstd | none
  .partitionBy("year", "month", "region")   // directory-based partitioning
  .parquet("s3://bucket/orders/")

// Partition cardinality guide:
// Too many partitions (high cardinality): millions of small files → slow metadata
// Too few partitions: large files → no pruning benefit
// Sweet spot: 1k–100k distinct values, target 256MB–1GB per partition directory
```

**Schema evolution:**
```scala
// After adding a new column to source files:
spark.read
  .option("mergeSchema", "true")
  .parquet("s3://bucket/orders/")
// New column appears as null for old files — handle with na.fill or coalesce
```

**TDD for Parquet sources:**
```scala
test("reads parquet and applies schema") {
  import java.nio.file.Files
  val tmpDir = Files.createTempDirectory("parquet-test").toString
  
  val data = Seq(Order(1L, 10L, 100.0, "active", "EU")).toDS()
  data.write.parquet(tmpDir)
  
  val result = spark.read.parquet(tmpDir).as[Order].collect()
  result should have length 1
  result.head.id shouldBe 1L
}
```

---

## Delta Lake

Delta adds ACID transactions, schema enforcement, and time travel to Parquet.

```scala
// Write
df.write
  .format("delta")
  .mode("overwrite")
  .partitionBy("year", "month")
  .save("s3://bucket/delta/orders/")

// Read
val df = spark.read.format("delta").load("s3://bucket/delta/orders/")

// Time travel — read historical versions
spark.read.format("delta").option("versionAsOf", 5).load("path/")
spark.read.format("delta").option("timestampAsOf", "2024-01-15").load("path/")

// Idempotent partition overwrite — only replaces matched partition
df.write
  .format("delta")
  .mode("overwrite")
  .option("replaceWhere", "year = 2024 AND month = 1")
  .save("s3://bucket/delta/orders/")

// Upsert (MERGE) — insert new rows, update existing
import io.delta.tables._

DeltaTable.forPath(spark, "s3://bucket/delta/orders/")
  .as("target")
  .merge(
    newData.as("source"),
    "target.id = source.id"
  )
  .whenMatched(condition = "source.updated_at > target.updated_at")
    .updateAll()
  .whenNotMatched()
    .insertAll()
  .execute()

// OPTIMIZE: compact small files (run periodically, not in every job)
DeltaTable.forPath("s3://bucket/delta/orders/").optimize().executeCompaction()

// VACUUM: delete old versions (default retention: 7 days)
DeltaTable.forPath("s3://bucket/delta/orders/").vacuum(retentionHours = 168)
```

**Streaming + batch unified (Delta as source and sink):**
```scala
// Write a stream to Delta
stream.writeStream
  .format("delta")
  .outputMode("append")
  .option("checkpointLocation", "s3://bucket/checkpoints/orders/")
  .start("s3://bucket/delta/orders/")

// Read Delta as a stream (detect new files automatically)
val deltaStream = spark.readStream.format("delta").load("s3://bucket/delta/orders/")
```

---

## JSON

```scala
// Read (define schema — never infer in production)
val df = spark.read
  .schema(orderSchema)
  .option("multiLine", "false")            // false = one JSON object per line (JSONL)
  .option("mode", "PERMISSIVE")            // PERMISSIVE | DROPMALFORMED | FAILFAST
  .option("columnNameOfCorruptRecord", "_corrupt_record")  // capture bad rows
  .json("s3://bucket/events/")

// Handle corrupt records
val clean  = df.filter($"_corrupt_record".isNull).drop("_corrupt_record")
val broken = df.filter($"_corrupt_record".isNotNull)
broken.write.text("s3://bucket/dlq/events/")  // dead letter queue

// Write
df.write
  .mode("append")
  .option("compression", "gzip")
  .json("s3://bucket/output/")

// Mode comparison:
// PERMISSIVE (default): bad rows get nulls + stored in _corrupt_record
// DROPMALFORMED: silently drop bad rows — dangerous, use carefully
// FAILFAST: throw exception on first bad row — good for strict validation
```

---

## CSV

Always define the schema explicitly. `inferSchema = true` requires two full passes over the data and picks the wrong type in ambiguous cases.

```scala
val df = spark.read
  .schema(schema)                        // always explicit — never inferSchema in prod
  .option("header", "true")
  .option("nullValue", "")              // what string represents null
  .option("dateFormat", "yyyy-MM-dd")
  .option("timestampFormat", "yyyy-MM-dd HH:mm:ss")
  .option("mode", "DROPMALFORMED")
  .csv("s3://bucket/exports/*.csv")
```

Use `DROPMALFORMED` only for CSV from legacy systems where malformed rows are expected noise. Route bad rows to a dead letter store when observability matters (see `error-handling.md`).

---

## JDBC

```scala
// Serial read (simple, single partition)
val df = spark.read
  .format("jdbc")
  .option("url", "jdbc:postgresql://host:5432/mydb")
  .option("dbtable", "orders")
  .option("user", sys.env("DB_USER"))
  .option("password", sys.env("DB_PASS"))
  .option("driver", "org.postgresql.Driver")
  .option("fetchsize", "10000")          // rows fetched per round-trip
  .load()

// Parallel read (partition by a numeric column — eliminates serial bottleneck)
val df = spark.read
  .format("jdbc")
  .option("url", jdbcUrl)
  .option("dbtable", "orders")
  .option("user", sys.env("DB_USER"))
  .option("password", sys.env("DB_PASS"))
  .option("partitionColumn", "id")       // must be numeric/date
  .option("lowerBound", "1")
  .option("upperBound", "10000000")
  .option("numPartitions", "50")         // 50 parallel DB connections — be careful
  .option("pushDownPredicate", "true")   // push WHERE clauses to DB
  .load()

// Predicate pushdown with custom query
val df = spark.read
  .format("jdbc")
  .option("url", jdbcUrl)
  .option("dbtable", "(SELECT * FROM orders WHERE status = 'active') t")
  .option("user", sys.env("DB_USER"))
  .option("password", sys.env("DB_PASS"))
  .load()

// Write to JDBC
df.write
  .format("jdbc")
  .option("url", jdbcUrl)
  .option("dbtable", "orders_summary")
  .option("user", sys.env("DB_USER"))
  .option("password", sys.env("DB_PASS"))
  .option("batchsize", "10000")          // rows per INSERT batch
  .mode("append")
  .save()
```

**numPartitions warning**: each partition opens a DB connection. Set numPartitions ≤ DB max_connections / 2.

---

## Kafka (Structured Streaming)

```scala
// Read from Kafka
val stream = spark.readStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "broker1:9092,broker2:9092")
  .option("subscribe", "orders")                           // single topic
  // .option("subscribePattern", "orders-.*")              // regex pattern
  // .option("assign", """{"orders":[0,1,2]}""")           // specific partitions
  .option("startingOffsets", "latest")                     // latest | earliest | JSON offsets
  .option("maxOffsetsPerTrigger", "100000")                // back-pressure: limit per micro-batch
  .option("kafka.security.protocol", "SASL_SSL")           // auth if needed
  .load()

// Parse the value (Kafka messages are binary)
val events = stream
  .selectExpr("CAST(key AS STRING)", "CAST(value AS STRING) as json",
               "topic", "partition", "offset", "timestamp")
  .select(from_json($"json", eventSchema).as("e"), $"timestamp")
  .select("e.*", "timestamp")

// Write to Kafka
result.writeStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "broker1:9092,broker2:9092")
  .option("topic", "order-events")
  .option("checkpointLocation", "s3://bucket/checkpoints/kafka-sink/")
  .start()

// Schema registry pattern (Confluent)
// Use spark-avro + schema registry client to decode/encode Avro messages
```

---

## Advanced I/O Patterns

**File size management:**
```scala
// After writing many small files, compact them
spark.read.parquet("s3://bucket/path/")
  .repartition(50)  // target ~200MB each
  .write.mode("overwrite").parquet("s3://bucket/path-compacted/")

// Control output file count
df.coalesce(10).write.parquet("s3://bucket/output/")  // max 10 files, no shuffle
```

**Parallel read from multiple paths:**
```scala
// Spark reads all paths in one call
val df = spark.read.parquet(
  "s3://bucket/orders/year=2024/month=01/",
  "s3://bucket/orders/year=2024/month=02/"
)

// Or with glob
val df = spark.read.parquet("s3://bucket/orders/year=2024/month=0[1-3]/")
```

**Compression codec comparison:**

| Codec | Speed | Ratio | Splittable | Use case |
|-------|-------|-------|-----------|----------|
| snappy | Fast | Moderate | Yes (Parquet) | Default, balanced |
| zstd | Medium | High | Yes (Parquet) | Better ratio, Spark 3+ |
| gzip | Slow write | High | No (for raw files) | Archival |
| lz4 | Fastest | Low | Yes | Speed-critical |
| none | Fastest | None | Yes | Debug only |

---

## UDFs — Last Resort Only

Always prefer built-in `functions._` over UDFs. Catalyst can optimize built-ins; UDFs are opaque black boxes — no predicate pushdown, no filter reordering, row-by-row JVM serialization.

```scala
import org.apache.spark.sql.functions._

// WRONG: UDF for a built-in function
val myUpper = udf((s: String) => s.toUpperCase)  // just use upper($"col")

// RIGHT: use built-in
df.withColumn("name_upper", upper($"name"))

// RIGHT: use from_json instead of a UDF for JSON parsing
val schema = new StructType().add("key", StringType).add("value", DoubleType)
df.withColumn("parsed", from_json($"raw_json", schema))
  .select("parsed.key", "parsed.value")

// When a UDF is genuinely unavoidable (custom business logic with no built-in equivalent):
val classifyCustomer = udf((amount: Double, orderCount: Long) =>
  if (amount > 10000 && orderCount > 20) "vip"
  else if (amount > 1000)                "regular"
  else                                   "new"
)

// Register for use in SQL strings too
spark.udf.register("classify_customer", classifyCustomer)

// Use in DataFrame API
df.withColumn("tier", classifyCustomer($"lifetime_amount", $"order_count"))

// Use in spark.sql()
spark.sql("SELECT id, classify_customer(lifetime_amount, order_count) AS tier FROM customers")

// Performance rules for unavoidable UDFs:
// 1. Never use a UDF in a filter if a built-in predicate can express it
// 2. Cache the DataFrame BEFORE applying a UDF (avoid recomputing expensive inputs)
// 3. Consider Pandas UDF (vectorized) for Python interop — operates on Arrow batches
//    not available in Scala natively; use SQL built-ins or Dataset.map instead
```

**Decision tree:**
```
Does a built-in function do it? → Use it (Catalyst-optimized)
  No → Can from_json / regexp_extract / transform / aggregate express it? → Use those
    No → Does it need row-by-row business logic? → Dataset.map on a case class (type-safe)
      No → UDF (last resort, document why)
```
