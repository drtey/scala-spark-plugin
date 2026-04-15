# Scala Collections Reference

## Choosing the Right Collection

| Use case | Collection | Why |
|----------|-----------|-----|
| Ordered sequence, mostly prepend | `List[A]` | O(1) head/prepend, O(n) indexed access |
| Ordered sequence, random access | `Vector[A]` | O(log n) all operations, effectively O(1) |
| Unordered unique values | `Set[A]` | O(1) lookup/add/remove |
| Key-value pairs | `Map[K,V]` | O(1) lookup |
| Fixed-size tuple of different types | `(A, B, C)` | Structural, type-safe |
| Streaming / lazy infinite sequences | `LazyList[A]` | Elements computed on demand |
| Mutable buffer (rare, avoid) | `ArrayBuffer[A]` | Only at boundaries; return immutable |

All above are **immutable** by default. Import `scala.collection.mutable` explicitly when you need mutable.

---

## List — Linked List

```scala
val xs: List[Int] = List(1, 2, 3, 4, 5)
val ys = 0 :: xs          // prepend: List(0,1,2,3,4,5) — O(1)
val zs = xs :+ 6          // append: List(1,2,3,4,5,6) — O(n), avoid
val head = xs.head         // 1 — O(1)
val tail = xs.tail         // List(2,3,4,5) — O(1)
val nth  = xs(2)           // 3 — O(n), avoid for large lists

// Pattern match on structure
xs match {
  case Nil       => "empty"
  case x :: Nil  => s"single element: $x"
  case x :: rest => s"head=$x, ${rest.length} more"
}

// Core operations
xs.map(_ * 2)                    // List(2,4,6,8,10)
xs.filter(_ % 2 == 0)            // List(2,4)
xs.foldLeft(0)(_ + _)            // 15
xs.foldRight(Nil: List[Int])(_ :: _)  // copy of list
xs.scanLeft(0)(_ + _)            // List(0,1,3,6,10,15) — running totals
xs.zip(List('a','b','c'))        // List((1,'a'),(2,'b'),(3,'c'))
xs.grouped(2).toList             // List(List(1,2), List(3,4), List(5))
xs.sliding(3).toList             // List(List(1,2,3), List(2,3,4), List(3,4,5))
xs.distinct                      // removes duplicates
xs.sorted                        // natural order
xs.sortBy(identity)(Ordering[Int].reverse)  // reverse
```

---

## Vector — Indexed Sequence

Prefer Vector over List when you need random access or frequent appends:

```scala
val v = Vector(1, 2, 3, 4, 5)
v(3)          // 4 — O(log n) ≈ O(1)
v :+ 6        // Vector(1,2,3,4,5,6) — O(log n)
v.updated(2, 99)  // Vector(1,2,99,4,5) — O(log n), immutable
```

---

## Map

```scala
val m: Map[String, Int] = Map("a" -> 1, "b" -> 2, "c" -> 3)

// Safe access — always use get or getOrElse
m.get("a")            // Some(1)
m.get("z")            // None
m.getOrElse("z", 0)  // 0
m("a")                // 1 — throws if missing, avoid

// Update (returns new Map)
val m2 = m + ("d" -> 4)           // add/update
val m3 = m - "b"                  // remove
val m4 = m ++ Map("d" -> 4, "e" -> 5)  // merge (right wins on conflict)

// Transform
m.mapValues(_ * 10)               // Map(a->10, b->20, c->30)
m.filterKeys(_ != "b")            // Map(a->1, c->3)
m.toList                          // List((a,1),(b,2),(c,3))

// GroupBy — very common pattern
val orders: List[Order] = loadOrders()
val byStatus: Map[String, List[Order]] = orders.groupBy(_.status)
val countByStatus: Map[String, Int]    = byStatus.view.mapValues(_.length).toMap
```

---

## Set

```scala
val s = Set(1, 2, 3, 4, 5)
s.contains(3)   // true — O(1)
s + 6           // Set(1,2,3,4,5,6)
s - 3           // Set(1,2,4,5)
s & Set(2,3,6)  // intersection: Set(2,3)
s | Set(6,7)    // union: Set(1,2,3,4,5,6,7)
s -- Set(2,4)   // difference: Set(1,3,5)
```

---

## LazyList — Infinite Sequences

```scala
// Infinite stream of natural numbers
val naturals: LazyList[Int] = LazyList.from(1)

naturals.take(5).toList     // List(1,2,3,4,5)
naturals.filter(_ % 2 == 0).take(5).toList  // List(2,4,6,8,10)

// Fibonacci — elegant with LazyList
val fibs: LazyList[BigInt] = {
  def go(a: BigInt, b: BigInt): LazyList[BigInt] = a #:: go(b, a + b)
  go(0, 1)
}
fibs.take(10).toList  // List(0,1,1,2,3,5,8,13,21,34)
```

---

## Common Collection Idioms

```scala
// Safe head/last
list.headOption   // Option[A]
list.lastOption   // Option[A]

// Flatten nested collections
List(List(1,2), List(3,4)).flatten   // List(1,2,3,4)

// flatMap = map + flatten
List(1,2,3).flatMap(n => List(n, n * 10))  // List(1,10,2,20,3,30)

// Collect — partial function map+filter
val mixed: List[Any] = List(1, "hello", 2, "world", 3)
mixed.collect { case n: Int => n * 2 }  // List(2,4,6)

// exists / forall / count
list.exists(_ > 5)    // any element satisfies?
list.forall(_ > 0)    // all elements satisfy?
list.count(_ % 2 == 0) // how many satisfy?

// mkString — join to string
List("a","b","c").mkString(", ")         // "a, b, c"
List("a","b","c").mkString("[", ", ", "]") // "[a, b, c]"

// tabulate — create by index
List.tabulate(5)(i => i * i)   // List(0,1,4,9,16)
Vector.tabulate(3)(i => s"item_$i")  // Vector(item_0, item_1, item_2)

// unfold — generate from state
LazyList.unfold(0)(n => if (n >= 5) None else Some((n, n + 1)))
// LazyList(0,1,2,3,4)
```

---

## groupMap and groupMapReduce (Scala 2.13+)

More efficient than `groupBy` + `mapValues` — single pass over the collection.

```scala
case class Person(name: String, city: String, age: Int)
val people = List(
  Person("Alice", "NYC", 30),
  Person("Bob", "LA", 25),
  Person("Carol", "NYC", 28)
)

// groupBy + mapValues — two passes
val old = people.groupBy(_.city).view.mapValues(_.map(_.name)).toMap

// groupMap — one pass: group by city, extract names
val namesByCity = people.groupMap(_.city)(_.name)
// Map("NYC" -> List("Alice", "Carol"), "LA" -> List("Bob"))

// groupMapReduce — one pass: group, extract, and aggregate
val avgAgeByCity = people.groupMapReduce(_.city)(_.age)(_ + _)
// Map("NYC" -> 58, "LA" -> 25) — sum of ages; divide by count for average

val countByCity = people.groupMapReduce(_.city)(_ => 1)(_ + _)
// Map("NYC" -> 2, "LA" -> 1)
```

---

## Collection Builder Pattern

Build mutable, return immutable — avoids N intermediate copies.

```scala
// Bad: creates a new Vector for every :+ (O(n log n) total allocations)
def processSlowly(input: List[Int]) =
  input.foldLeft(Vector.empty[Int]) { (acc, n) =>
    if (n > 0) acc :+ (n * 2) else acc
  }

// Good: one allocation at the end
def process(input: List[Int]) = {
  val builder = Vector.newBuilder[Int]
  input.foreach { n =>
    if (n > 0) builder += n * 2
  }
  builder.result()  // immutable Vector, single allocation
}

// ListBuffer — efficient for lists
def buildList(input: Seq[Int]) = {
  val buf = scala.collection.mutable.ListBuffer.empty[Int]
  input.foreach { n => if (n % 2 == 0) buf += n * 3 }
  buf.toList  // O(1) conversion
}

// StringBuilder — for string accumulation
def buildReport(items: Seq[String]) = {
  val sb = new StringBuilder
  items.foreach { item =>
    sb.append(item).append('\n')
  }
  sb.toString
}
```

---

## Specialized Collections for Performance

```scala
// ArraySeq — backed by Java array, O(1) random access, primitives unboxed
import scala.collection.immutable.ArraySeq

val arr = ArraySeq(1, 2, 3, 4, 5)
arr(3)           // O(1), no boxing for Int
arr.map(_ * 2)   // ArraySeq(2,4,6,8,10)

// When to use: known-size sequence, frequent random access, numeric data

// SortedMap — maintains key order, O(log n) operations
import scala.collection.immutable.SortedMap

val sorted = SortedMap("banana" -> 2, "apple" -> 1, "cherry" -> 3)
sorted.keys.toList    // List("apple", "banana", "cherry") — always sorted
sorted.firstKey       // "apple"
sorted.lastKey        // "cherry"
sorted.range("b", "c") // SortedMap("banana" -> 2) — range query

// When to use: ordered iteration is required, range queries

// SortedSet — ordered unique values
import scala.collection.immutable.SortedSet

val scores = SortedSet(85, 92, 78, 95, 88)
scores.max  // 95
scores.min  // 78
scores.range(80, 90)  // SortedSet(85, 88)

// Queue — efficient functional FIFO
import scala.collection.immutable.Queue

val q  = Queue(1, 2, 3)
val (head, rest) = q.dequeue   // (1, Queue(2, 3))
val q2 = rest.enqueue(4)       // Queue(2, 3, 4)
```

---

## Parallel Collections (Scala 2 / 3 via scala-parallel-collections)

```scala
import scala.collection.parallel.CollectionConverters._

val data = (1 to 1_000_000).toVector

// Sequential
val result = data.map(expensiveCompute)

// Parallel — uses ForkJoinPool, splits work across cores
val resultPar = data.par.map(expensiveCompute).seq

// When to use:
// Independent operations, CPU-bound, large collections
// Do NOT use with I/O or shared mutable state — race conditions
```

---

## Performance Summary

| Operation | List | Vector | Map/Set |
|-----------|------|--------|---------|
| head/last | O(1)/O(n) | O(log n) | N/A |
| prepend   | O(1) | O(log n) | N/A |
| append    | O(n) | O(log n) | N/A |
| index     | O(n) | O(log n) | N/A |
| lookup    | N/A  | N/A      | O(1) |
| insert    | N/A  | N/A      | O(1) |
| iterate   | O(n) | O(n)     | O(n) |
