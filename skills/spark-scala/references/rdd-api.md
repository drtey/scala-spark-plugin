# RDD Patterns

## When to Use RDD Instead of DataFrame

Use RDD only when:
- You need a custom `Partitioner` for domain-specific partitioning logic
- Working with raw binary data or non-tabular formats
- Complex accumulation logic requiring a custom `AccumulatorV2`
- Operations not expressible in the structured API (`mapPartitionsWithIndex`, `pipe`)
- Interoperating with pre-Spark 2 code

**Default to DataFrames.** Catalyst and Tungsten optimizations do not apply to RDDs. Convert back to DataFrame/Dataset as soon as possible.

---

## Partition-Level Operations

Use `mapPartitions` when setup per element is expensive (DB connections, JSON parsers, model loading). Amortize the cost over the whole partition.

```scala
// Wrong — opens and closes a DB connection for every row
rdd.map(row => { val conn = openConnection(); conn.lookup(row) })

// Right — one connection per partition
rdd.mapPartitions { iter =>
  val conn = openConnection()
  try iter.map(row => conn.lookup(row))
  finally conn.close()
}

// mapPartitionsWithIndex — when the partition index matters
rdd.mapPartitionsWithIndex { (idx, iter) =>
  iter.map(row => (idx, row))
}
```

---

## Pair RDD Aggregations

### reduceByKey over groupByKey

`groupByKey` shuffles all `(K, V)` pairs before combining. `reduceByKey` performs a map-side combine first, drastically reducing shuffle data.

```scala
// Wrong — shuffles every value across the network, then sums
pairs.groupByKey().mapValues(_.sum)

// Right — partial sums computed per partition, only aggregates shuffled
pairs.reduceByKey(_ + _)
```

### combineByKey for complex aggregations

Use when the accumulator type differs from the value type (e.g., computing per-key averages).

```scala
// Per-key average: values are Int, combiner is (sum, count)
val avgByKey = pairs.combineByKey(
  createCombiner = (v: Int) => (v, 1),
  mergeValue     = (acc: (Int, Int), v: Int) => (acc._1 + v, acc._2 + 1),
  mergeCombiners = (a: (Int, Int), b: (Int, Int)) => (a._1 + b._1, a._2 + b._2)
).mapValues { case (sum, count) => sum.toDouble / count }
```

---

## Repartitioning

```scala
rdd.repartition(200)  // full shuffle — use to increase OR decrease partition count
rdd.coalesce(10)      // no shuffle — use ONLY to reduce partition count
```

Use `coalesce` after a `filter` that shrinks the dataset significantly. Use `repartition` when increasing parallelism or rebalancing skewed partitions.

---

## Accumulators

Accumulators aggregate values across tasks. Read them on the driver only after the action completes.

```scala
val counter = sc.longAccumulator("row-count")
rdd.foreach(row => counter.add(1))
println(counter.value)  // read after action
```

**Custom accumulator** — extend `AccumulatorV2[IN, OUT]` when the accumulation type is complex:

```scala
class SetAccumulator extends AccumulatorV2[String, Set[String]] {
  private var _set = Set.empty[String]

  def isZero = _set.isEmpty
  def copy()  = { val a = new SetAccumulator; a._set = _set; a }
  def reset() = _set = Set.empty
  def add(v: String) = _set += v
  def merge(other: AccumulatorV2[String, Set[String]]) =
    _set ++= other.asInstanceOf[SetAccumulator]._set
  def value = _set
}

val uniqueErrors = new SetAccumulator
sc.register(uniqueErrors, "unique-error-codes")
logRdd.foreach(line => if (line.contains("ERROR")) uniqueErrors.add(line.split(" ")(2)))
println(uniqueErrors.value)  // read after action
```

**Caveats:**
- Only accurate after a successful action on a non-streaming query
- In streaming, accumulators may overcount if a micro-batch is retried — do not use for auditable counts in streaming

---

## Checkpointing (Lineage Truncation)

Truncates long RDD lineage chains to prevent driver OOM and speed up failure recovery. Not the same as streaming checkpoints.

```scala
sc.setCheckpointDir("s3://bucket/rdd-checkpoints/")

val rdd = computeWithLongChain(...)
rdd.checkpoint()
rdd.count()  // must materialize to trigger the checkpoint write
// subsequent operations read from disk, not from the full lineage
```

Use when:
- Iterative algorithms (the lineage grows with each iteration)
- Transformation chains longer than ~20 stages
- Source is non-replayable (socket streams, volatile external data)

---

## Performance Rules

1. **`reduceByKey` over `groupByKey`** — map-side combine dramatically reduces shuffle size
2. **`mapPartitions` for setup-heavy ops** — amortize connection/parser cost over the whole partition
3. **`combineByKey` for complex per-key aggregation** — most general and most efficient option
4. **`coalesce` over `repartition` when reducing** — avoids an unnecessary full shuffle
5. **Cache sparingly** — `rdd.cache()` only if you reuse it; always `rdd.unpersist()` when done
6. **Convert to DataFrame as soon as possible** — RDDs bypass Catalyst and Tungsten

---

## TDD for RDD Code

Extract row-level logic into pure functions and test them without Spark first:

```scala
def parseLogLine(line: String): Option[(String, Int)] =
  line.split(" ") match {
    case Array(_, _, errorCode, count) => Some((errorCode, count.toInt))
    case _                             => None
  }

test("parseLogLine extracts error code and count") {
  parseLogLine("2024-01-01 ERROR E404 15") shouldBe Some(("E404", 15))
  parseLogLine("malformed line")           shouldBe None
}

// Integration test with local SparkContext
test("error counts aggregated correctly") {
  val counts = sc.parallelize(Seq(
    "2024-01-01 ERROR E404 1",
    "2024-01-01 ERROR E404 1",
    "2024-01-01 ERROR E500 1"
  ))
    .flatMap(parseLogLine)
    .reduceByKey(_ + _)
    .collectAsMap()

  counts("E404") shouldBe 2
  counts("E500") shouldBe 1
}
```
