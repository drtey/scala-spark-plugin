# Error Handling in Spark

## Handling Corrupt or Invalid Data at Read Time

Never silently drop bad data. Make failures visible by capturing invalid rows and routing them to a dead letter path.

```scala
// Wrong — DROPMALFORMED silently discards rows with no trace
spark.read
  .option("mode", "DROPMALFORMED")
  .json("s3://bucket/events/")

// Right — PERMISSIVE captures corrupt rows in a dedicated column
val df = spark.read
  .schema(eventSchema)
  .option("mode", "PERMISSIVE")
  .option("columnNameOfCorruptRecord", "_corrupt_record")
  .json("s3://bucket/events/")

val clean  = df.filter($"_corrupt_record".isNull).drop("_corrupt_record")
val broken = df.filter($"_corrupt_record".isNotNull)

broken
  .select($"_corrupt_record".as("raw"))
  .write.mode("append")
  .json("s3://bucket/dlq/events/")
```

**Mode comparison:**

| Mode | Behavior | Use when |
|------|----------|----------|
| `PERMISSIVE` | Nulls on bad fields, stores row in `_corrupt_record` | Default — captures everything |
| `DROPMALFORMED` | Silently discards bad rows | Never in production |
| `FAILFAST` | Throws exception on first bad row | Strict batch pipelines where any corruption is fatal |

---

## Dead Letter Pattern

Every pipeline that ingests external data must have a dead letter sink. It is the only way to know what was lost.

```scala
case class ParseResult[A](good: Dataset[A], bad: Dataset[String])

def parseEvents(raw: Dataset[String]): ParseResult[Event] = {
  val attempted = raw.map { line =>
    scala.util.Try(parseJson(line)) match {
      case scala.util.Success(event) => (Some(event), None)
      case scala.util.Failure(e)     => (None, Some(s"$line\t${e.getMessage}"))
    }
  }

  ParseResult(
    good = attempted.flatMap(_._1),
    bad  = attempted.flatMap(_._2)
  )
}

val result = parseEvents(rawLines)
result.good.write.mode("append").parquet("s3://bucket/events/")
result.bad.write.mode("append").text("s3://bucket/dlq/events/")
```

Size the DLQ, monitor it, and alert when it grows. A growing DLQ is a signal that the schema or the source is changing.

---

## Idempotent Writes

Write operations must be safe to retry. A job that fails midway and restarts must not produce duplicates or partial results.

```scala
// Wrong — append accumulates data on every retry
df.write.mode("append").parquet("s3://bucket/output/")

// Right — overwrite the whole partition atomically
df.write
  .mode("overwrite")
  .partitionBy("date", "region")
  .parquet("s3://bucket/output/")

// Right — Delta replaceWhere for surgical idempotent partition replacement
df.write
  .format("delta")
  .mode("overwrite")
  .option("replaceWhere", "date = '2024-01-15'")
  .save("s3://bucket/delta/events/")

// Right — Delta MERGE for upsert idempotency
DeltaTable.forPath(spark, "s3://bucket/delta/orders/")
  .as("target")
  .merge(
    newBatch.as("source"),
    "target.id = source.id AND target.date = source.date"
  )
  .whenMatched().updateAll()
  .whenNotMatched().insertAll()
  .execute()
```

---

## Classifying Errors: Recoverable vs Fatal

Establish a policy before writing the job. Different error types demand different responses.

| Error type | Example | Response |
|------------|---------|----------|
| Corrupt input row | Malformed JSON | Dead letter — continue processing |
| Missing expected partition | Date partition not found | Alert + fail job |
| Schema mismatch | Column type changed upstream | Fail job — do not silently coerce |
| Transient I/O | S3 timeout, JDBC refused | Retry with backoff |
| Downstream locked | Delta table locked | Fail fast — do not queue indefinitely |

```scala
// Retry transient failures; propagate permanent ones
@tailrec
def writeWithRetry(df: DataFrame, path: String, attempts: Int = 3): Unit =
  scala.util.Try(df.write.mode("overwrite").parquet(path)) match {
    case scala.util.Success(_)         => ()
    case scala.util.Failure(_) if attempts > 1 => writeWithRetry(df, path, attempts - 1)
    case scala.util.Failure(e)         => throw e
  }
```

---

## Checkpoint and Recovery in Streaming

Checkpoints give Structured Streaming exactly-once semantics — but only when used correctly.

```scala
// Always set checkpointLocation — without it, every restart reprocesses from scratch
stream.writeStream
  .format("delta")
  .option("checkpointLocation", "s3://bucket/checkpoints/order-stream/")
  .outputMode("append")
  .start("s3://bucket/delta/orders/")
```

Rules:
- **One checkpoint per query** — sharing a checkpoint between two queries corrupts offset tracking for both
- **Never delete a checkpoint unless you intend to reprocess from the beginning**
- **Change your transformation logic → new checkpoint location** — the old checkpoint tracks offsets for the old logic

**Batch backfill using the streaming engine** — processes all available data, then stops cleanly:

```scala
stream.writeStream
  .trigger(Trigger.AvailableNow)
  .option("checkpointLocation", "s3://bucket/checkpoints/backfill-v2/")
  .format("delta")
  .start("s3://bucket/delta/events/")
  .awaitTermination()
```

**What checkpoints do NOT protect against:**
- Schema changes in the source data
- Bugs in your transformation logic (fix the logic, restart with a new checkpoint)
- Output sink failures (guaranteed by the sink's own idempotency, not by Spark)
