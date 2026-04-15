# data-skills-plugin

Claude Code plugin with two expert skills for Scala and Apache Spark development.

## Skills

### `spark-scala`
Expert guidance for Apache Spark development in Scala. Covers:
- SparkSession setup and configuration
- DataFrame vs Dataset API (with type-safe case classes)
- Transformations, actions, and lazy evaluation
- Schema definition and data sources (Parquet, Delta, JDBC, Kafka)
- Aggregations and window functions
- Join strategies (broadcast, sort-merge, skew handling)
- Partitioning, caching, and checkpointing
- RDDs and when to use them
- Structured Streaming with watermarks and stateful operations
- UDFs and when to avoid them
- Error handling: dead letter pattern, idempotent writes, corrupt data
- Clean transformation design and testing

**Reference docs:**
- `references/optimization.md` — AQE, Catalyst, shuffle tuning, anti-patterns
- `references/streaming-patterns.md` — watermarking, stateful ops, exactly-once
- `references/data-sources.md` — format selection, Parquet/Delta/JDBC/Kafka patterns
- `references/error-handling.md` — dead letter pattern, idempotent writes, checkpoint recovery
- `references/rdd-api.md` — when to use RDDs, PairRDD aggregation patterns, accumulators
- `references/spark-ml.md` — MLlib pipelines, transformers, estimators, evaluation

### `scala`
Expert guidance for idiomatic, functional Scala. Covers:
- Pure functions, immutability, referential transparency
- Domain modeling: case classes, sealed trait ADTs, smart constructors
- Avoiding primitive obsession with opaque types
- Pattern matching (type, structure, guards, nested)
- Error handling: Option, Either, Try, ValidatedNel (no exceptions in pure code)
- Higher-order functions and immutable collections
- For comprehensions as monadic composition
- Tail recursion with `@tailrec`
- Type classes and given/using (Scala 3)
- Concurrency: Futures, ExecutionContext, Cats Effect IO
- Clean Code and Clean Architecture principles applied to Scala

**Reference docs:**
- `references/fp-patterns.md` — Functor, Monad, Monoid, IO monad, Validated, State
- `references/collections.md` — collection hierarchy, performance guide, idioms
- `references/type-system.md` — variance, bounds, higher-kinded types, opaque types
- `references/domain-modeling.md` — case classes, sealed traits/ADTs, smart constructors
- `references/concurrency-futures.md` — Futures, ExecutionContext, Cats Effect IO patterns
- `references/testing.md` — TDD, ScalaTest, property-based testing, test doubles
- `references/clean-architecture.md` — layers, dependency rule, use cases, ports & adapters
- `references/code-smells.md` — Scala-specific smells, refactoring recipes

## Installation

```bash
/plugin install https://github.com/drtey/scala-spark-plugin
```

## Structure

```
data-skills-plugin/
├── .claude-plugin/
│   └── plugin.json
└── skills/
    ├── spark-scala/
    │   ├── SKILL.md
    │   └── references/
    │       ├── optimization.md
    │       ├── streaming-patterns.md
    │       ├── data-sources.md
    │       ├── error-handling.md
    │       ├── rdd-api.md
    │       └── spark-ml.md
    └── scala/
        ├── SKILL.md
        └── references/
            ├── fp-patterns.md
            ├── collections.md
            ├── type-system.md
            ├── domain-modeling.md
            ├── concurrency-futures.md
            ├── testing.md
            ├── clean-architecture.md
            └── code-smells.md
```
