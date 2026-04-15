# Spark Optimization Reference

## Catalyst Optimizer — Four Phases

```
Phase 1: Analysis
  — Resolves column names against the catalog
  — Binds to the actual DataFrame schema
  — Type-checks all expressions
  — Result: Resolved Logical Plan

Phase 2: Logical Optimization
  — Predicate pushdown (move filters as close to source as possible)
  — Column pruning (drop unreferenced columns early)
  — Constant folding: replace lit(2) + lit(3) with lit(5)
  — Boolean simplification: x == x → true
  — Dead code elimination
  — Result: Optimized Logical Plan
  — Key insight: a .filter() written after a .groupBy() gets automatically
    pushed BEFORE the groupBy by Catalyst — write for clarity, let Catalyst optimize

Phase 3: Physical Planning
  — Chooses join strategy (broadcast vs sort-merge vs shuffle-hash)
  — Determines shuffle partitions
  — Exploits bucketed tables to skip shuffle
  — Generates multiple candidate physical plans, picks cheapest using CBO if enabled
  — Result: Physical Plan (what .explain() shows)

Phase 4: Code Generation (Tungsten)
  — Compiles physical plan to JVM bytecode
  — Whole-stage code gen: fuses multiple operators into a single compiled method
  — Eliminates virtual dispatch overhead between operators
  — Result: Compiled native code running directly on JVM
```

```scala
// Inspect each phase
df.explain("extended")   // shows all four plans
df.explain("formatted")  // cleaner tree format (Spark 3+)
df.explain("cost")       // with row count / size estimates
```

---

## Tungsten — Binary Execution Engine

```
1. Binary row format
   — Stores rows as compact binary, not Java objects
   — No object header overhead (12–16 bytes per Java object saved)
   — Enables direct byte comparison for sorting/hashing

2. Off-heap memory management
   — Uses sun.misc.Unsafe to allocate memory outside Java heap
   — No GC pressure for intermediate row data
   — Avoids stop-the-world GC pauses during shuffle

3. Whole-stage code generation
   — Fuses filter + project + aggregate into a single tight loop
   — Eliminates virtual method dispatch between operators
   — Result: code resembles hand-written nested loops

Active by default in Spark 2+. No configuration required.
```

```scala
// Verify whole-stage code gen is active
spark.conf.get("spark.sql.codegen.wholeStage")  // should be "true"

// Disable only for debugging (never in production)
spark.conf.set("spark.sql.codegen.wholeStage", "false")
```

---

## OOM Diagnosis — Flowchart

```
SYMPTOM                            ROOT CAUSE                    FIX
───────────────────────────────────────────────────────────────────────────
"GC overhead limit exceeded"   →  working set > executor heap  →  increase --executor-memory
 (executor logs)                                                   OR reduce spark.sql.shuffle.partitions

"Shuffle Spill (Disk) > 0"     →  sort buffer < shuffle data   →  increase --executor-memory
 (Stages tab, UI)                                                  OR increase shuffle partitions

One executor OOM,              →  data skew: one partition      →  enable AQE skew join
 others fine                      is disproportionately large      OR salt the skewed key

"OutOfMemoryError: Java heap"  →  broadcast table too large    →  lower autoBroadcastJoinThreshold
 during broadcast                                                  OR disable broadcast for that join

Driver OOM                     →  df.collect() on large data   →  use df.write instead of collect
                                  OR count() materializes         use foreachBatch for streaming
                                  huge accumulator
```

```scala
// Diagnose broadcast OOM
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "10mb")  // lower from default 10mb
// Or disable broadcast entirely for a specific join
df1.join(df2.hint("merge"), "key")  // force sort-merge, no broadcast
```

---

## Partition Size Formula

```scala
// Target: 100–300 MB per post-shuffle partition
// Formula: partitions = total_shuffled_data_MB / 200

// Step 1: measure total data volume
df.cache().count()
// Check Storage tab → "Size in Memory" → total dataset size

// Step 2: set shuffle partitions accordingly
// e.g., 40 GB dataset → 40000 / 200 = 200 partitions
spark.conf.set("spark.sql.shuffle.partitions", "200")

// Step 3: verify partition sizes after a wide transformation
val sizes = df.rdd.mapPartitions(it => Iterator(it.size)).collect()
println(s"min=${sizes.min} max=${sizes.max} mean=${sizes.sum / sizes.length}")
// Tasks finishing in < 10ms → too many partitions (reduce, recompute from Step 2)
// Tasks running > 2 min → too few partitions (increase)

// AQE handles this automatically in Spark 3.2+:
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.adaptive.advisoryPartitionSizeInBytes", "200mb")
```

---

## Adaptive Query Execution (AQE) — Spark 3+

AQE re-optimizes the query plan at runtime using actual shuffle statistics — not estimates.

```scala
// Enable (default in Spark 3.2+)
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")

// AQE capabilities:
// 1. Dynamic partition coalescing: merges small post-shuffle partitions automatically
// 2. Skew join handling: splits skewed partitions, replicates matching side
// 3. Runtime join strategy switching: downgrades sort-merge → broadcast if one side
//    becomes small after filtering (even if it wasn't small at plan time)
```

AQE is NOT a substitute for good design — it fixes edge cases, not fundamentally bad plans.

---

## Reading and Acting on Query Plans

```scala
df.explain()            // physical plan
df.explain("extended")  // logical + optimized logical + physical
df.explain("cost")      // with row count / size estimates
df.explain("formatted") // cleaner tree format (Spark 3+)
```

**Key operators to recognize:**

| Operator | Meaning | Action needed |
|----------|---------|---------------|
| `Exchange` | Shuffle — data moves across network | Minimize these |
| `BroadcastHashJoin` | Broadcast join — good | Verify expected side was broadcast |
| `SortMergeJoin` | Shuffle-based join | OK for large+large; check for skew |
| `CartesianProduct` | Cross join — almost always wrong | Check join condition |
| `FileScan` | Reading data | Check `PushedFilters` — are your filters pushed? |
| `HashAggregate` | Two-stage: partial then final | Normal for groupBy |
| `LocalTableScan` | Reading in-memory (broadcast) | Good |

**Spot a bad plan in tests:**
```scala
// Assert broadcast was chosen (no shuffle on either side)
val planStr = df.queryExecution.executedPlan.toString
assert(planStr.contains("BroadcastHashJoin"), "Expected broadcast join")
assert(!planStr.contains("SortMergeJoin"), "Unexpected shuffle join")

// Assert filters were pushed down to file scan
assert(planStr.contains("PushedFilters: [IsNotNull(status), EqualTo(status,active)]"))
```

---

## Spark UI — Interpreting Performance Data

Open at `http://driver-host:4040` during job execution.

**Jobs tab:**
- Find the stage that took longest — that's your bottleneck
- Failed stages indicate OOM or data issues

**Stages tab (most important):**
- **Task duration**: if max >> median → data skew. A single slow task blocks the whole stage
- **Shuffle Read/Write**: large numbers = heavy shuffle. Can you eliminate a groupBy?
- **GC time**: if > 10% of task time → executor memory pressure. Increase `executor-memory` or reduce `spark.memory.fraction` for storage

**Storage tab:**
- Cache hit ratio: if `Fraction Cached` < 100% → cache thrashing. Reduce what you cache
- `Memory Deserialized` vs `Memory Serialized` — deserialized is faster access

**SQL tab:**
- View the physical plan as a visual DAG
- Click on a node to see row counts, timing, and size estimates
- Estimates vs actual: large gaps indicate stale statistics (run `ANALYZE TABLE`)

**Executors tab:**
- `GC Time` column — flag executors with > 10% GC (memory pressure)
- `Shuffle Spill (Disk)` — data spilled to disk = insufficient executor memory for shuffle

---

## Shuffle Tuning

```scala
// Number of output partitions after a shuffle (default: 200)
spark.conf.set("spark.sql.shuffle.partitions", "400")

// Rule of thumb: target 100–200 MB per output partition
// partitions = total_shuffled_data_MB / 150

// Measure actual partition sizes after a wide transformation
val sizes = df.rdd.mapPartitions(it => Iterator(it.size)).collect()
println(s"min=${sizes.min} max=${sizes.max} avg=${sizes.sum / sizes.length}")

// Avoid shuffle by pre-partitioning your writes:
df.write.partitionBy("region", "date").parquet("s3://bucket/data/")
// Then filter on those columns → Spark does local reads, no shuffle
```

---

## Cost-Based Optimizer (CBO)

CBO uses table statistics to choose better join strategies and orderings.

```scala
spark.conf.set("spark.sql.cbo.enabled", "true")
spark.conf.set("spark.sql.statistics.histogram.enabled", "true")

// After creating or updating a table, compute statistics
spark.sql("ANALYZE TABLE orders COMPUTE STATISTICS")
spark.sql("ANALYZE TABLE orders COMPUTE STATISTICS FOR ALL COLUMNS")
spark.sql("ANALYZE TABLE orders PARTITION (year=2024) COMPUTE STATISTICS")
```

Without `ANALYZE`, Spark uses rough heuristics for row count estimates — leading to wrong join strategy choices.

---

## Dynamic Allocation

```scala
spark.conf.set("spark.dynamicAllocation.enabled", "true")
spark.conf.set("spark.dynamicAllocation.minExecutors", "2")
spark.conf.set("spark.dynamicAllocation.maxExecutors", "50")
spark.conf.set("spark.dynamicAllocation.initialExecutors", "5")
spark.conf.set("spark.dynamicAllocation.executorIdleTimeout", "60s")
spark.conf.set("spark.dynamicAllocation.cachedExecutorIdleTimeout", "300s")
// Note: when using caching, executors holding cached data are kept longer
```

Use dynamic allocation in shared clusters (YARN/Kubernetes). Avoid it for jobs with strict SLAs where startup latency matters.

---

## Data Skew — Diagnosis and Solutions

**Diagnose skew:**
1. Stages tab → look for one task taking 10× longer than others
2. One executor OOMing while others have memory to spare
3. Shuffle read: one task reading GB while others read MB

**Solution 1: AQE skew join (automatic in Spark 3+)**
```scala
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionFactor", "5")   // 5× median
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes", "256MB")
```

**Solution 2: Salting (manual, for extreme skew)**
```scala
val n = 10  // salt factor — tune based on skew severity

val saltedOrders = orders
  .withColumn("salt", (rand() * n).cast("int"))
  .withColumn("join_key_salted", concat($"skewed_key", lit("_"), $"salt"))

val expandedLookup = lookup
  .withColumn("salt", explode(array((0 until n).map(lit(_)): _*)))
  .withColumn("join_key_salted", concat($"skewed_key", lit("_"), $"salt"))

saltedOrders.join(expandedLookup, "join_key_salted").drop("salt", "join_key_salted")
```

**Solution 3: Isolate the hot key**
```scala
// Separate the dominant key and join it differently
val hotKey = "customer_XYZ"
val hotOrders     = orders.filter($"customer_id" === hotKey)
val regularOrders = orders.filter($"customer_id" =!= hotKey)

val hotResult     = hotOrders.join(broadcast(customers.filter($"id" === hotKey)), "customer_id")
val regularResult = regularOrders.join(customers, "customer_id")

hotResult.union(regularResult)
```

---

## Caching Strategy and Storage Levels

```scala
import org.apache.spark.storage.StorageLevel

// ── Storage Level Reference ───────────────────────────────────────────────────────
//
//  Level                │ In Memory │ On Disk │ Serialized │ Notes
//  ─────────────────────┼───────────┼─────────┼────────────┼──────────────────────
//  MEMORY_ONLY          │    yes    │   no    │     no     │ Default .cache(). Fastest
//                       │           │         │            │ access. Evicted partitions
//                       │           │         │            │ are recomputed, not spilled.
//  MEMORY_ONLY_SER      │    yes    │   no    │     yes    │ Smaller footprint than
//                       │           │         │            │ MEMORY_ONLY; higher CPU cost
//                       │           │         │            │ for serialization/deserialization.
//  MEMORY_AND_DISK      │    yes    │   yes   │     no     │ Spills to disk when RAM is
//                       │           │         │            │ full. No recomputation needed.
//                       │           │         │            │ Best balance for most jobs.
//  MEMORY_AND_DISK_SER  │    yes    │   yes   │     yes    │ Serialized in RAM + disk.
//                       │           │         │            │ Less RAM, more CPU.
//  DISK_ONLY            │    no     │   yes   │     yes    │ Slowest. For data that is
//                       │           │         │            │ accessed rarely but too large
//                       │           │         │            │ to recompute.
//  OFF_HEAP             │ off-JVM   │   no    │     yes    │ Bypasses GC entirely.
//                       │           │         │            │ Requires:
//                       │           │         │            │ spark.memory.offHeap.enabled=true

// Usage
df.cache()                                        // = MEMORY_ONLY (shorthand)
df.persist(StorageLevel.MEMORY_AND_DISK)          // explicit level
df.persist(StorageLevel.MEMORY_AND_DISK_SER)      // explicit serialized

// Decision guide
// Dataset fits in RAM, reused many times             → MEMORY_ONLY
// MEMORY_ONLY causing GC pauses (Executors tab > 10%)→ MEMORY_ONLY_SER
// Dataset too large for RAM, reused across actions   → MEMORY_AND_DISK
// Very large dataset, RAM extremely limited           → MEMORY_AND_DISK_SER
// Rarely accessed, recomputing is expensive           → DISK_ONLY
// GC is a bottleneck even with serialization          → OFF_HEAP

// Always materialize after persisting, always unpersist when done
val cached = expensiveJoinResult.persist(StorageLevel.MEMORY_AND_DISK)
cached.count()      // triggers computation and populates cache
val result1 = cached.agg(sum("amount")).collect()
val result2 = cached.orderBy(desc("amount")).limit(10).collect()
cached.unpersist()  // release executor memory
```

**When to cache:**
- Dataset is used in 2+ actions in the same job
- Computation is expensive (complex joins, many aggregations, ML iterations)
- Dataset is reused by multiple downstream branches of the DAG

**When NOT to cache:**
- Dataset is used only once — caching adds overhead with no benefit
- Dataset is very large relative to executor memory — cache thrashing degrades everything
- The source is fast (e.g., a small Parquet file with column pruning)

---

## Broadcast Variables (non-DataFrame)

```scala
// For non-DataFrame data shared across all tasks in all executors
val lookupMap: Map[String, Double] = loadExchangeRates()
val rates = spark.sparkContext.broadcast(lookupMap)

val converted = df.map { row =>
  val rate = rates.value.getOrElse(row.currency, 1.0)
  row.copy(amountUSD = row.amount * rate)
}

// Unpersist when done — frees executor memory
rates.unpersist()
```

---

## Memory Configuration

```scala
// spark-submit flags:
// --driver-memory 4g
// --executor-memory 8g
// --executor-cores 4

// Fraction of executor memory for execution + storage (default: 0.6)
spark.conf.set("spark.memory.fraction", "0.8")

// Within the above fraction, reserved for storage/cache (default: 0.5)
spark.conf.set("spark.memory.storageFraction", "0.5")

// Unified memory model (Spark 1.6+): execution and storage share the same pool.
// Storage can evict execution pages and vice versa. No fixed boundary.

// Reduce overhead for jobs with many small objects:
spark.conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
spark.conf.set("spark.kryoserializer.buffer.max", "512m")
```

**Memory pressure signals:**
- GC time > 10% in Executors tab
- `Shuffle Spill (Disk)` > 0 in Stages tab
- `java.lang.OutOfMemoryError: GC overhead limit exceeded` in logs

**Response:** increase `--executor-memory`, reduce `spark.sql.shuffle.partitions`, or add more executors.

---

## Predicate Pushdown and Column Pruning

Catalyst automatically pushes filters and column selections to the data source.

```scala
// Both pushdown optimizations apply here:
spark.read.parquet("s3://bucket/orders/")
  .select("order_id", "amount", "status")   // column pruning: reads only 3 of N columns
  .filter($"status" === "active")            // predicate pushdown: skips non-active files
  .filter($"amount" > 100)                   // pushed into Parquet page-level index

// Verify in query plan:
df.explain()
// Look for: PushedFilters: [IsNotNull(status), EqualTo(status,active), GreaterThan(amount,100.0)]
```

**Pushdown fails when:**
- Filter is on a computed column (apply filter before `withColumn`)
- UDF is in the filter (UDFs are opaque to Catalyst)
- Filter column type mismatches the schema type

---

## Anti-Patterns

```scala
// BAD: collect() on large data → driver OOM
val all = df.collect()

// GOOD: write to storage, process distributed
df.write.parquet("s3://bucket/output/")

// BAD: repartition before write just to get 1 file
df.repartition(1).write.csv("output/")  // full shuffle

// GOOD: coalesce instead (no shuffle)
df.coalesce(1).write.csv("output/")

// BAD: multiple actions on uncached DataFrame (recomputes)
val n    = df.count()    // job 1 — reads and processes all data
val rows = df.take(10)   // job 2 — reads and processes all data again

// GOOD: cache if reusing, or combine into one action
val cached = df.cache()
val n    = cached.count()
val rows = cached.take(10)
cached.unpersist()

// BAD: UDF for built-in function
val myUpper = udf((s: String) => s.toUpperCase)  // bypasses Catalyst

// GOOD: use built-in
import org.apache.spark.sql.functions.upper
df.withColumn("name", upper($"name"))

// BAD: Cartesian product (accidental cross join)
df1.join(df2)  // no join condition → cross join

// GOOD: always specify join condition
df1.join(df2, df1("id") === df2("ref_id"))
```
