# Scala Type System Reference

## Type Inference

Annotate only at public API boundaries or when inference picks the wrong type.

```scala
// Inferred — prefer this in implementation code
val n         = 42
val name      = "Alice"
val items     = List(1, 2, 3)
val lookup    = Map("a" -> 1, "b" -> 2)
val maybeUser = Option(findUser(id))

// Annotate at public API boundaries (documents the contract)
def processOrder(id: Long): Either[String, Order] = ???

// Annotate when inference picks the wrong type
val zero = 0L          // without L, inferred as Int
val rate = 0.5f        // without f, inferred as Double

// Let the compiler propagate through transformations
val doubled   = items.map(_ * 2)            // List[Int]
val filtered  = items.filter(_ > 1)         // List[Int]
val asStrings = items.map(_.toString)       // List[String]
```

---

## Variance

Variance controls the subtyping relationship between parameterized types.

### Covariance (+A) — Producers

If `Cat <: Animal`, then `Container[Cat] <: Container[Animal]`.

```scala
sealed trait Animal
case class Cat(name: String) extends Animal
case class Dog(name: String) extends Animal

// Covariant container — can only produce A, never consume
sealed trait Container[+A]
case class Box[+A](value: A) extends Container[A]

val catBox: Box[Cat] = Box(Cat("Whiskers"))
val animalBox: Box[Animal] = catBox  // OK — covariance allows this

// Use covariance for read-only producers
sealed trait Source[+A] {
  def get: A
}

// Scala built-ins that are covariant: List[+A], Option[+A], Vector[+A]
val cats: List[Cat]     = List(Cat("a"), Cat("b"))
val animals: List[Animal] = cats   // works — List is covariant
```

### Contravariance (-A) — Consumers

If `Cat <: Animal`, then `Consumer[Animal] <: Consumer[Cat]`.

```scala
// Contravariant consumer — can consume A, never produce it
trait Printer[-A] {
  def print(a: A): String
}

val animalPrinter: Printer[Animal] = (a: Animal) => a.toString
val catPrinter: Printer[Cat] = animalPrinter  // OK — contravariance

// Function1[-A, +B] is contravariant in input, covariant in output
// This is why: f: Animal => String can replace f: Cat => String
val describeAnimal: Animal => String = _.toString
val describeCat: Cat => String = describeAnimal  // works
```

### Invariance — Read and Write

No subtyping relationship between parameterized types.

```scala
// Array[A] is invariant — allows both reads and writes
val cats: Array[Cat]     = Array(Cat("a"))
// val animals: Array[Animal] = cats  // compile error — invariant!

// Mutable containers must be invariant for type safety
class Queue[A](private var items: List[A]) {
  def enqueue(a: A): Unit = { items = items :+ a }
  def dequeue: Option[A]  = items.headOption
}
// Queue[Cat] cannot be assigned to Queue[Animal] — correct!
```

### Use-site Variance (Wildcards)

```scala
// When working with invariant containers, use use-site variance
def printAll[A](source: java.util.List[_ <: A]): Unit =
  source.forEach(item => println(item))

// Equivalent Scala approach with bounds
def readFrom[A, B <: A](source: Queue[B]): Option[A] =
  source.dequeue
```

---

## Type Bounds

### Upper Bounds — restrict to subtypes of A

```scala
// A must be Animal or a subtype
def describe[A <: Animal](a: A) = s"Animal: ${a.getClass.getSimpleName}"

describe(Cat("Whiskers"))  // OK
describe(Dog("Rex"))       // OK
// describe(42)            // compile error — Int is not Animal

// Upper bound enables using methods from the bound type
def makeNoise[A <: Animal](animals: List[A]) =
  animals.map(_.getClass.getSimpleName)
```

### Lower Bounds — restrict to supertypes of A

```scala
// Useful in covariant containers when you need to insert
sealed trait Container[+A] {
  // Without lower bound: def prepend(a: A) — illegal, A is covariant
  // With lower bound:   allow inserting a supertype
  def prepend[B >: A](b: B): Container[B]
}

// List.:: is defined this way:
// def ::[B >: A](x: B): List[B]
val cats: List[Cat] = List(Cat("a"))
val animals = (Dog("Rex"): Animal) :: cats  // List[Animal] — lower bound in action
```

### Combined Bounds

```scala
// A must be a supertype of Cat and a subtype of Animal
def process[A >: Cat <: Animal](a: A): String = a.getClass.getSimpleName
```

---

## Higher-Kinded Types

Types parameterized by type constructors (types that take a type parameter).

```scala
// F[_] means: F takes one type parameter
// Enables abstraction over containers (List, Option, Future, IO, etc.)

trait Mappable[F[_]] {
  def map[A, B](fa: F[A])(f: A => B): F[B]
}

// Instances
val listMappable = new Mappable[List] {
  def map[A, B](fa: List[A])(f: A => B) = fa.map(f)
}
val optionMappable = new Mappable[Option] {
  def map[A, B](fa: Option[A])(f: A => B) = fa.map(f)
}

// Write code generic over any F[_] that has a map
def doubleAll[F[_]: Mappable](fa: F[Int]) =
  implicitly[Mappable[F]].map(fa)(_ * 2)

doubleAll(List(1, 2, 3))(listMappable)     // List(2, 4, 6)
doubleAll(Option(5))(optionMappable)        // Some(10)

```

---

## Phantom Types

Types that exist only at compile time — carry zero runtime cost but prevent illegal states.

```scala
// State machine enforced at compile time
sealed trait Sealed
sealed trait Open

class Door[State] private (val id: Int)

object Door {
  def create(id: Int): Door[Open] = new Door(id)
}

def close(door: Door[Open]): Door[Sealed]  = new Door(door.id)
def open(door: Door[Sealed]): Door[Open]   = new Door(door.id)

val door    = Door.create(1)    // Door[Open]
val closed  = close(door)       // Door[Sealed]
val opened  = open(closed)      // Door[Open]
// open(door)                   // compile error — cannot open an already-open door

// Units of measure — prevent mixing incompatible values
sealed trait Meters
sealed trait Feet

case class Length[Unit](value: Double) extends AnyVal

def addLengths[U](a: Length[U], b: Length[U]) = Length[U](a.value + b.value)

val m1: Length[Meters] = Length(100.0)
val m2: Length[Meters] = Length(50.0)
val f1: Length[Feet]   = Length(30.0)

addLengths(m1, m2)  // OK
// addLengths(m1, f1)  // compile error — type mismatch
```

---

## Path-Dependent Types

A type that depends on a specific *instance* of another type. Each instance has its own inner type.

```scala
class Database {
  class Connection  // each Database instance has its own Connection type

  def connect(): Connection = new Connection
  def query(conn: Connection): String = "result"  // conn must be THIS database's connection
}

val db1 = new Database
val db2 = new Database

val conn1 = db1.connect()  // db1.Connection
val conn2 = db2.connect()  // db2.Connection

db1.query(conn1)  // OK
// db1.query(conn2)  // compile error — db2.Connection is not db1.Connection

// Practical: Graph with type-safe node membership
class Graph {
  class Node(val label: String)
  def addEdge(a: Node, b: Node): Unit = ???  // only nodes from THIS graph
}
```

---

## Union and Intersection Types (Scala 3)

### Union Types — A or B

```scala
// A | B: value can be either A or B
def describe(value: Int | String): String = value match {
  case n: Int    => s"number: $n"
  case s: String => s"text: $s"
}

describe(42)       // "number: 42"
describe("hello")  // "text: hello"

// Better than using a sealed trait for simple cases
def parseId(raw: String): Long | String =
  raw.toLongOption.getOrElse(raw)
```

### Intersection Types — A and B

```scala
// A & B: value must satisfy both types simultaneously
trait Readable { def read(): String }
trait Writable { def write(s: String): Unit }

// A value that is both Readable and Writable
def copy(from: Readable, to: Readable & Writable): Unit =
  to.write(from.read())

// More expressive than requiring a third trait ReadWrite extends Readable, Writable
```

---

## Opaque Types (Scala 3)

Type aliases with zero boxing overhead. The alias is only transparent inside the defining object.

```scala
object Domain {
  opaque type UserId  = Long
  opaque type OrderId = Long
  opaque type Email   = String

  object UserId {
    def apply(id: Long): UserId = id
    extension (id: UserId) def value: Long = id
  }

  object Email {
    def parse(raw: String): Option[Email] =
      Option.when(raw.contains("@"))(raw)
    extension (e: Email) def value: String = e
  }
}

import Domain._

val userId  = UserId(42L)
val orderId = OrderId(99L)
// userId == orderId   // compile error — different opaque types!

// Zero boxing: UserId is Long at runtime — no allocation
val email = Email.parse("user@example.com")  // Option[Email]
```

**Compare with value classes** (`extends AnyVal`): opaque types are strictly enforced within the file boundary; value classes can be bypassed via `asInstanceOf`. Prefer opaque types in Scala 3.

---

## Structural Types (Scala 3)

Duck typing at compile time. A type is valid if it has the required structure.

```scala
import scala.reflect.Selectable.reflectiveSelectable

// Accepts anything with a .name: String field
def greet(entity: { val name: String }): String = s"Hello, ${entity.name}"

case class User(name: String, email: String)
case class Product(name: String, price: Double)

greet(User("Alice", "a@b.com"))  // "Hello, Alice"
greet(Product("Widget", 9.99))   // "Hello, Widget"

// Note: uses reflection at runtime — prefer type classes for performance-critical code
```

---

## Context Bounds (Type Class Constraints)

Shorthand for implicit parameters in generic code.

```scala
// Long form
def serialize[A](a: A)(implicit enc: Encoder[A]): String = enc.encode(a)

// Context bound form (equivalent)
def serialize[A: Encoder](a: A): String = implicitly[Encoder[A]].encode(a)

// Scala 3 — summon replaces implicitly
def serialize[A: Encoder](a: A): String = summon[Encoder[A]].encode(a)

// Multiple constraints
def process[A: Encoder: Decoder: Show](a: A): String = summon[Show[A]].show(a)

// Real example with Spark
import org.apache.spark.sql.Encoder
def toDataset[A: Encoder](items: Seq[A])(implicit spark: SparkSession) =
  spark.createDataset(items)
```

---

## Type Aliases

Name complex types to improve readability.

```scala
// Simple alias — transparent (no new type)
type UserId   = Long
type ErrorMsg = String
type Validated[A] = Either[List[ErrorMsg], A]

// Use in signatures
def validate(id: UserId): Validated[User] = ???

// Parameterized alias
type Result[A]    = Either[String, A]
type EitherNel[E, A] = Either[List[E], A]

// Common in Cats
type ValidatedNel[E, A] = cats.data.Validated[cats.data.NonEmptyList[E], A]
```

---

## Existential Types / Wildcards

```scala
// When you know a type exists but don't care what it is
def printLength(list: List[?]): Int = list.length  // Scala 3 syntax

// Scala 2
def printLength(list: List[_]): Int = list.length

// Practical: heterogeneous collection by shared trait
def renderAll(widgets: List[Renderable]): List[String] =
  widgets.map(_.render)
```
