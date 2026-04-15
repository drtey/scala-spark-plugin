# Structured Streaming Patterns

## Core Model

```
Input stream → micro-batch → [unbounded table] → transformation → output sink
                              (same Dataset API as batch)
```

---

## Event Time vs Processing Time

**Processing time**: when Spark receives the event. Unreliable — network delays mean events arrive out of true order.

**Event time**: the timestamp embedded in the data itself (when the event actually occurred). Use for all analytical streaming jobs. Handle late arrivals with `withWatermark`.

```
Real world:           [Ecuador event: T=10:00] [Virginia event: T=10:00]
Arrives at Spark:     [Virginia: T=10:00] ........ [Ecuador: T=10:00, delay=5min]
Processing time sees: Virginia first → wrong ordering
Event time sees:      Both at 10:00 → correct
```

---

## Trigger Modes

| Trigger | Behavior | Use case |
|---------|----------|---------|
| `ProcessingTime("30 seconds")` | Fire every N seconds | Continuous streaming with latency SLA |
| `Once()` | Process all available data, stop | Scheduled batch-streaming hybrid |
| `AvailableNow()` | Like Once but multi-batch (Spark 3.3+) | Large backfills, scheduled runs |
| `Continuous("1 second")` | Experimental, millisecond latency | Ultra-low latency (limited aggregation support) |

```scala
// Micro-batch: fire every 30 seconds
.trigger(Trigger.ProcessingTime("30 seconds"))

// Once: run exactly one batch then stop — great for hourly cron jobs
// Uses checkpoint to know where to start, processes new data, exits
.trigger(Trigger.Once())

// AvailableNow: process all available data in multiple batches (respects maxFilesPerTrigger)
.trigger(Trigger.AvailableNow())

// Continuous (experimental): sub-second latency, no aggregations
.trigger(Trigger.Continuous("100 milliseconds"))
```

---

## Basic Streaming Pipeline

```scala
val spark = SparkSession.builder().appName("OrderStream").getOrCreate()
import spark.implicits._

// Source
val raw = spark.readStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "broker:9092")
  .option("subscribe", "orders")
  .option("startingOffsets", "latest")
  .option("maxOffsetsPerTrigger", "50000")  // back-pressure
  .load()

// Parse
val events = raw
  .selectExpr("CAST(value AS STRING) as json", "timestamp as kafka_ts")
  .select(from_json($"json", orderSchema).as("o"), $"kafka_ts")
  .select("o.*", "kafka_ts")

// Transform (same as batch)
val aggregated = events
  .withWatermark("event_time", "10 minutes")
  .groupBy(window($"event_time", "5 minutes"), $"region")
  .agg(count("*").as("orders"), sum($"amount").as("revenue"))

// Sink
val query = aggregated.writeStream
  .format("delta")
  .outputMode("append")
  .trigger(Trigger.ProcessingTime("30 seconds"))
  .option("checkpointLocation", "s3://bucket/checkpoints/order-stream/")
  .start("s3://bucket/delta/order-aggregates/")

query.awaitTermination()
```

---

## Type-Safe Streaming with Dataset

```scala
case class Flight(dest: String, origin: String, count: BigInt)
case class RouteSummary(route: String, totalFlights: BigInt)

val schema = Encoders.product[Flight].schema

val flightStream: Dataset[Flight] = spark.readStream
  .schema(schema)
  .parquet("s3://bucket/flights/")   // reads new files as they appear
  .as[Flight]                         // type-safe from here

// Type-safe filter — compiler catches field name mistakes
val filteredStream: Dataset[Flight] =
  flightStream.filter(f => f.origin != f.dest)

// Type-safe aggregation
val routeCounts = filteredStream
  .map(f => RouteSummary(s"${f.origin}->${f.dest}", f.count))
  .groupByKey(_.route)
  .reduceGroups((a, b) => a.copy(totalFlights = a.totalFlights + b.totalFlights))
  .map(_._2)

routeCounts.writeStream
  .format("memory")
  .queryName("route_counts")
  .outputMode("complete")
  .start()
```

---

## Output Modes

| Mode | Description | Use with |
|------|-------------|---------|
| `append` | Only new rows emitted | Windowed aggregations with watermark, no-aggregation queries |
| `complete` | Full result table every trigger | Global aggregations (stateful), small result sets |
| `update` | Only changed rows | Aggregations without watermark (experimental for most sinks) |

```scala
// Append: emit only when a window is finalized (past watermark boundary)
.outputMode("append")  // requires withWatermark on aggregation

// Complete: re-emit the entire aggregation table each trigger
.outputMode("complete")  // works without watermark; state grows unboundedly

// Update: emit only rows that changed
.outputMode("update")  // supported by console, memory, Delta sinks
```

---

## Watermarks — Choosing the Delay

```scala
// Watermark = how long to wait for late data before closing a window
// Spark tracks: max(event_time seen) - watermark_delay = current watermark
// Windows with end_time < current_watermark are finalized and emitted (in append mode)

events.withWatermark("event_time", "15 minutes")
// → waits 15 minutes for late data
// → a 10:00–10:05 window is finalized at 10:20 (5min window end + 15min watermark)

// Choosing the delay:
// Too short → late data dropped (incorrect results)
// Too long → more state in memory, higher latency to see results
// Typical: measure P99 lateness of your data; set watermark ≥ that value

// Window types
events.groupBy(window($"event_time", "10 minutes"))         // tumbling: non-overlapping
events.groupBy(window($"event_time", "10 minutes", "5 minutes"))  // sliding: overlapping
```

**Session windows** (Spark 3.2+):
```scala
import org.apache.spark.sql.functions.session_window

events
  .withWatermark("event_time", "10 minutes")
  .groupBy(session_window($"event_time", "5 minutes"), $"userId")
  .agg(count("*").as("events"), sum($"amount").as("session_revenue"))
// Session closes when no event for userId for 5 minutes
```

---

## Stateful Processing — mapGroupsWithState vs flatMapGroupsWithState

**mapGroupsWithState**: emits **exactly one row per group per trigger** — always produces output.
**flatMapGroupsWithState**: emits **0..N rows per group per trigger** — use for delayed output (session end).

```scala
import org.apache.spark.sql.streaming.{GroupState, GroupStateTimeout, OutputMode}

// ── mapGroupsWithState: real-time running totals (one row per group, every trigger) ──
case class UserMetrics(userId: String, eventCount: Long, totalAmount: Double)

val runningMetrics = events
  .groupByKey(_.userId)
  .mapGroupsWithState(
    timeoutConf = GroupStateTimeout.ProcessingTimeTimeout()
  ) { (userId: String, newEvents: Iterator[UserEvent], state: GroupState[UserMetrics]) =>

    if (state.hasTimedOut) {
      state.remove()
      UserMetrics(userId, 0L, 0.0)  // final zero row on timeout
    } else {
      val prev    = state.getOption.getOrElse(UserMetrics(userId, 0L, 0.0))
      val batch   = newEvents.toList
      val updated = UserMetrics(userId, prev.eventCount + batch.length,
                                prev.totalAmount + batch.map(_.amount).sum)
      state.update(updated)
      state.setTimeoutDuration("10 minutes")
      updated  // emit one row per group per trigger
    }
  }

// ── flatMapGroupsWithState: session detection (emit 0..N rows per trigger) ──
case class UserEvent(userId: String, eventType: String, timestamp: Long, amount: Double)
case class UserSession(userId: String, events: List[String], startTime: Long, lastSeen: Long)
case class SessionAlert(userId: String, message: String)

val alerts: Dataset[SessionAlert] = events
  .withWatermark("event_time", "30 minutes")
  .groupByKey(_.userId)
  .flatMapGroupsWithState(
    outputMode = OutputMode.Append(),
    timeoutConf = GroupStateTimeout.EventTimeTimeout()
  ) { (userId: String, newEvents: Iterator[UserEvent], state: GroupState[UserSession]) =>

    if (state.hasTimedOut) {
      val session = state.get
      state.remove()
      Iterator(SessionAlert(userId, s"Session ended: ${session.events.length} events"))
    } else {
      val current  = state.getOption.getOrElse(
        UserSession(userId, Nil, System.currentTimeMillis(), 0L)
      )
      val allEvents = newEvents.toList
      val updated   = current.copy(
        events   = current.events ++ allEvents.map(_.eventType),
        lastSeen = allEvents.map(_.timestamp).max
      )
      state.update(updated)
      state.setTimeoutTimestamp(updated.lastSeen + 30 * 60 * 1000L)
      Iterator.empty  // no output until timeout fires
    }
  }
```

---

## Stream-Stream Joins

Both streams **must** have watermarks. The join condition **must** include a time-range predicate — otherwise state grows unboundedly.

```scala
// Both streams need watermarks so Spark knows when to evict old state
val clicks = clickStream
  .withWatermark("clickTime", "30 minutes")

val orders = orderStream
  .withWatermark("orderTime", "30 minutes")

// Inner join: requires time-range predicate to bound state
val attributed = clicks.join(
  orders,
  expr("""
    clicks.userId = orders.userId AND
    clicks.clickTime >= orders.orderTime AND
    clicks.clickTime <= orders.orderTime + INTERVAL 1 HOUR
  """),
  "inner"
)
// State for clicks: kept for (watermark + 1 hour) after clickTime
// State for orders: kept for watermark after orderTime

// Left outer join — rows with no match emit with nulls for right side
// Both watermarks must advance far enough to know no match is possible
val withNulls = clicks.join(
  orders,
  expr("""
    clicks.userId = orders.userId AND
    clicks.clickTime BETWEEN orders.orderTime AND orders.orderTime + INTERVAL 1 HOUR
  """),
  "left_outer"
)
// Output mode must be Append for stream-stream joins
```

**State retention rule:** each side's state is kept as long as the current watermark minus the event time is within the join time range. Without the time-range predicate, all state is kept forever → OOM.

---

## Continuous Processing (Experimental)

Sub-millisecond latency for pure map/filter pipelines. No aggregations, no stateful ops.

```scala
// Standard micro-batch (default): 100ms–30s latency
stream.writeStream
  .format("kafka")
  .trigger(Trigger.ProcessingTime("5 seconds"))
  .option("checkpointLocation", checkpointPath)
  .start()

// Continuous processing: ~1ms latency, higher resource usage
stream.writeStream
  .format("kafka")
  .trigger(Trigger.Continuous("100 milliseconds"))  // checkpoint interval, not trigger interval
  .option("checkpointLocation", checkpointPath)
  .start()

// Restrictions (as of Spark 3.x):
// - Only map/filter/flatMap operations supported
// - No groupBy, no aggregations, no stateful operations
// - No joins
// - Use case: ultra-low latency for simple transformations (log enrichment, event routing)
```

---

## Streaming Progress Metrics

```scala
val query = stream.writeStream.start()

val p = query.lastProgress

// Throughput
p.inputRowsPerSecond   // rows arriving per second at the source
p.processedRowsPerSecond  // rows processed per second by Spark
// If input >> processed consistently → backlog growing → add executors or increase trigger interval

// Micro-batch timing (milliseconds)
p.durationMs.get("triggerExecution")  // total time for this trigger
p.durationMs.get("addBatch")          // time actually processing data
p.durationMs.get("queryPlanning")     // Catalyst planning time (should be < 100ms)
p.durationMs.get("getBatch")          // time reading from source
// If addBatch >> triggerExecution → transformation is the bottleneck
// If getBatch >> addBatch → source is the bottleneck (slow read, not enough partitions)

// State store (for stateful queries)
p.stateOperators(0).numRowsTotal      // rows currently in state store
p.stateOperators(0).memoryUsedBytes   // state store memory
p.stateOperators(0).numRowsDroppedByWatermark  // rows evicted by watermark
// Growing numRowsTotal + shrinking numRowsDroppedByWatermark → watermark too conservative

// Monitoring pattern: alert on throughput degradation
spark.streams.addListener(new StreamingQueryListener {
  override def onQueryStarted(e: QueryStartedEvent): Unit = ()
  override def onQueryProgress(e: QueryProgressEvent): Unit = {
    val p = e.progress
    if (p.inputRowsPerSecond > p.processedRowsPerSecond * 1.2)
      log.warn(s"Stream ${p.name} falling behind: " +
               s"in=${p.inputRowsPerSecond} out=${p.processedRowsPerSecond}")
  }
  override def onQueryTerminated(e: QueryTerminatedEvent): Unit =
    e.exception.foreach(ex => log.error(s"Stream failed: $ex"))
})
```

---

## Exactly-Once Guarantees

Exactly-once semantics require all three:

1. **Source is replayable**: Kafka (yes), files (yes), sockets (no)
2. **Sink is idempotent or transactional**: Delta (yes, transactional), Kafka (yes, idempotent), JDBC with upsert (yes), JDBC with append (at-least-once)
3. **Checkpoint is durable**: S3 or HDFS (yes), local disk (no — lost on restart)

```scala
// Checkpoint = exactly-once + fault tolerance
.option("checkpointLocation", "s3://bucket/checkpoints/unique-job-name/")

// Rules:
// 1. Each streaming query needs its own unique checkpoint path
// 2. Delete checkpoint to reset offsets (loses all state)
// 3. Never share a checkpoint between different queries
// 4. Checkpoint on durable storage (S3, HDFS) — not local disk
```

---

## foreachBatch — Custom Sink Logic

```scala
// Write to multiple sinks or use non-streaming APIs
aggregated.writeStream
  .foreachBatch { (batchDF: DataFrame, batchId: Long) =>
    // batchDF is a static DataFrame — use all static APIs

    // Upsert to Delta (idempotent)
    DeltaTable.forPath("s3://bucket/delta/summaries/")
      .as("target")
      .merge(batchDF.as("source"), "target.key = source.key")
      .whenMatched().updateAll()
      .whenNotMatched().insertAll()
      .execute()

    // Also write to PostgreSQL
    batchDF.write
      .format("jdbc")
      .option("url", jdbcUrl)
      .option("dbtable", "order_summaries")
      .option("user", sys.env("DB_USER"))
      .option("password", sys.env("DB_PASS"))
      .mode("append")
      .save()
  }
  .option("checkpointLocation", checkpointPath)
  .start()
```

---

## Monitoring and Testing Streams

**Monitoring:**
```scala
val query = aggregated.writeStream.start()

// Last micro-batch statistics
query.lastProgress
// {
//   "id": "...",
//   "inputRowsPerSecond": 12453.0,
//   "processedRowsPerSecond": 10234.0,
//   "durationMs": { "addBatch": 450, "queryPlanning": 12, ... }
// }

query.recentProgress.foreach(println)

// Add a listener for alerting
spark.streams.addListener(new StreamingQueryListener {
  override def onQueryStarted(e: QueryStartedEvent): Unit = log.info(s"Started: ${e.id}")
  override def onQueryProgress(e: QueryProgressEvent): Unit = {
    val rps = e.progress.inputRowsPerSecond
    if (rps < 1000) alert(s"Low throughput: $rps rows/sec")
  }
  override def onQueryTerminated(e: QueryTerminatedEvent): Unit =
    e.exception.foreach(ex => alert(s"Stream FAILED: $ex"))
})
```

**TDD for streaming:**
```scala
// Use memory sink for unit testing — no external dependencies
val query = events.writeStream
  .format("memory")
  .queryName("test_output")
  .outputMode("append")
  .start()

query.processAllAvailable()  // blocks until all available data is processed

val result = spark.sql("SELECT * FROM test_output").collect()
result should have length 3
result.map(_.getAs[String]("region")).toSet shouldBe Set("EU", "US")

query.stop()
```

**Test Trigger.Once in tests:**
```scala
// Trigger.Once is perfect for testing: process available data, stop
val query = stream.writeStream
  .trigger(Trigger.Once())
  .format("delta")
  .option("checkpointLocation", tmpCheckpoint)
  .start(tmpOutput)

query.awaitTermination()  // blocks until Once trigger completes

spark.read.format("delta").load(tmpOutput).count() shouldBe expectedRows
```

---

## DStream API — Legacy Reference

Use **Structured Streaming** for all new development. DStream API is found in pre-Spark-2.x codebases.

```scala
import org.apache.spark.streaming.{StreamingContext, Seconds}

val ssc = new StreamingContext(spark.sparkContext, batchDuration = Seconds(5))

// Source — create a DStream from Kafka (legacy direct API)
import org.apache.spark.streaming.kafka010._
val stream = KafkaUtils.createDirectStream[String, String](
  ssc,
  LocationStrategies.PreferConsistent,
  ConsumerStrategies.Subscribe[String, String](Set("orders"), kafkaParams)
)
```

### Stateless Transformations

Each batch is processed independently — no memory of previous batches.

```scala
stream.map(record => record.value().toUpperCase)        // one-to-one
stream.flatMap(record => record.value().split(","))     // one-to-many
stream.filter(record => record.value().contains("EU")) // keep matching
stream.reduceByKey(_ + _)                               // aggregate by key within batch
```

### Stateful Transformations

State is maintained across batches. Requires a checkpoint directory.

```scala
ssc.checkpoint("s3://bucket/checkpoints/dstream/")

// updateStateByKey — accumulate state across all batches
val pairs = stream.map(r => (r.key(), 1))
val runningCounts = pairs.updateStateByKey {
  (newValues: Seq[Int], previousState: Option[Int]) =>
    Some(previousState.getOrElse(0) + newValues.sum)
}

// mapWithState — more efficient, supports timeouts (Spark 1.6+)
val spec = StateSpec.function {
  (key: String, value: Option[Int], state: State[Int]) =>
    val updated = state.getOption.getOrElse(0) + value.getOrElse(0)
    state.update(updated)
    (key, updated)
}
pairs.mapWithState(spec)
```

### Window Operations

```scala
// window(windowLength, slideInterval) — combine batches over a time window
stream.window(Seconds(30), Seconds(10))

// reduceByKeyAndWindow — efficient windowed aggregation with inverse function
stream
  .map(r => (r.key(), 1))
  .reduceByKeyAndWindow(
    reduceFunc  = _ + _,         // forward function
    invReduceFunc = _ - _,       // inverse: removes old data efficiently
    windowDuration = Seconds(30),
    slideDuration  = Seconds(10)
  )
```

### Receivers

```scala
// Reliable receiver: sends ACK to the source only after data is replicated.
//   Guarantees at-least-once delivery.
// Unreliable receiver: no ACK (e.g., socket stream). May lose data on failure.

// Default persistence for network input streams: replicated across 2 nodes
stream.persist()  // MEMORY_AND_DISK_SER_2 by default for network streams

ssc.start()
ssc.awaitTermination()
```

### Migration Path: DStream → Structured Streaming

| DStream API | Structured Streaming equivalent |
|---|---|
| `updateStateByKey` | `mapGroupsWithState` |
| `mapWithState` | `mapGroupsWithState` / `flatMapGroupsWithState` |
| `window()` | `withWatermark` + `window($"event_time", ...)` |
| `foreachRDD` | `foreachBatch` |
| `DStream.transform` | `Dataset.transform` |
| `StreamingContext` | `SparkSession.readStream` |
| Receiver-based sources | Built-in connectors (Kafka, files) |
