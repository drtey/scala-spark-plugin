---
name: scala
description: "Expert guidance for writing idiomatic, functional, test-driven Scala code. Use this skill when the user asks to write Scala code, design Scala types, handle errors without exceptions, use Option/Either/Try/Validated, write pure functions, model domains with case classes and sealed traits, use pattern matching, higher-order functions, for comprehensions, type classes, implicits/given, variance, by-name parameters, multiple parameter groups, recursion, or any Scala language feature. Also use when the user asks about functional programming in Scala: immutability, referential transparency, monads, functors, composing functions, IO monad, or TDD in Scala. Trigger even if the user just says \"write this in Scala\", \"how do I do X in Scala\", or \"help me design this Scala type\"."
---

# Idiomatic Scala — Functional, Clean, and Test-Driven

---

## The 5 FP Rules

> 1. **No null values** — use `Option` instead.
> 2. **Only pure functions** — output depends only on input; no side effects.
> 3. **Only immutable values** — `val`, never `var`. Immutable collections by default.
> 4. **Every `if` must have an `else`** — use expressions, not statements. Everything returns a value.
> 5. **Data structures + pure functions** — not OOP classes that mix data and behavior.

---

## Three Programming Paradigms

```
Structured programming (Dijkstra, 1960s)
  Removes: unrestricted GOTO (jumps anywhere)
  Replaces with: if/while/for (structured control flow)
  Enables: proofs of correctness, testable modules

Object-oriented programming (1970s)
  Removes: unrestricted function pointers (for polymorphism)
  Replaces with: safe dynamic dispatch (sealed traits, virtual methods)
  Enables: dependency inversion — plug in any implementation at the call site

Functional programming (Church, 1930s)
  Removes: mutable variable assignment
  Replaces with: immutable values, pure functions
  Enables: referential transparency — a call is always substitutable by its result
```

Scala uses all three deliberately:
- FP core: pure functions, immutable data, `val`, case classes
- OOP seams: `sealed trait` + pattern matching for extensible type hierarchies
- Structured control: `for` comprehensions, `if` expressions, `match`

---

## FP as Algebra — The Key Insight

Pure functions are algebraic equations. Composing them is algebraic substitution.

```scala
// Each line is an algebraic equation:
val emailDoc  = getEmailFromServer(src)    // b = f(a)
val emailAddr = getAddr(emailDoc)           // c = g(b)
val domain    = getDomainName(emailAddr)    // d = h(c)

// Because all functions are pure, these are equivalent:
val domain = getDomainName(getAddr(getEmailFromServer(src)))  // d = h(g(f(a)))
```

---

## Scala Type Hierarchy

```
                        Any
                       /   \
                  AnyRef   AnyVal
                (reference) (value)
                  /            \
           String, List,      Int  Long  Double  Float
           case classes,      Char  Boolean  Short  Byte
           objects...                              Unit
                   \                              /
                              Nothing
                     (subtype of everything — bottom type)
```

```scala
// Any — root of the entire type hierarchy
// Has: == != equals hashCode toString
val x: Any = 42         // any value fits
val y: Any = "hello"    // any value fits

// AnyRef — equivalent to java.lang.Object; all classes and objects
val s: AnyRef = "string"       // String extends AnyRef
val l: AnyRef = List(1, 2, 3)  // List extends AnyRef

// AnyVal — value types; compiled to JVM primitives, no heap allocation
val n: AnyVal = 42       // Int  — 32-bit signed integer
val d: AnyVal = 3.14     // Double — 64-bit floating point
val b: AnyVal = true     // Boolean
val u: AnyVal = ()       // Unit — the "void" of Scala

// Unit — return type for functions that exist only for their side effects
def log(msg: String): Unit = println(msg)  // returns ()
// A function returning Unit has only side effects — it cannot be pure

// Nothing — the bottom type: subtype of every type, no value exists
// Used as return type for functions that never complete normally
def fail(msg: String): Nothing = throw new RuntimeException(msg)
def loop(): Nothing = { while (true) {}; ??? }

// Consequence: Nothing fits anywhere a type is expected
val x: Int = if (valid) 42 else fail("bad input")  // compiles: Nothing <: Int
val opt: Option[String] = if (found) Some("ok") else fail("missing")
```

---

## Null, Nil, None, Nothing — The Four "Empty" Concepts

```scala
// Null — the null reference. Type: Null (subtype of all AnyRef types).
// Never use null in new Scala code — it crashes on dereference.
val s: String = null         // compiles, but s.length throws NullPointerException
// Interop: when calling Java APIs that may return null, wrap immediately:
val safe: Option[String] = Option(javaMethod())  // None if null, Some(value) otherwise

// Nil — the empty List. Object of type List[Nothing].
// Because Nothing <: A for any A, Nil works as List[A] for any A.
val empty: List[Int]    = Nil
val empty2: List[String] = Nil
Nil == List()            // true
1 :: Nil                 // List(1)
Nil.isEmpty              // true

// None — the empty Option. Object of type Option[Nothing].
// Use instead of null for values that may be absent.
val absent: Option[Int] = None
None.map(_ + 1)          // None — safe, no NPE
None.getOrElse(0)        // 0

// Nothing — the bottom type. No value of type Nothing can ever exist.
// Appears as return type for code that always throws or never terminates.
// The Scala predef defines: def ???: Nothing (unimplemented method marker)
def notYet: Int = ???    // compiles: ??? returns Nothing, Nothing <: Int

// Summary table
//
//  Concept    │ Type         │ Has a value? │ Use for
//  ───────────┼──────────────┼──────────────┼──────────────────────────────────
//  null       │ Null         │ yes (null)   │ Java interop only — avoid otherwise
//  Nil        │ List[Nothing]│ yes (empty)  │ empty List literal
//  None       │ Option[Noth.]│ yes (empty)  │ absent optional value
//  Nothing    │ Nothing      │ no           │ diverging computation, ??? stubs
```

---

## object vs class — Singletons and Companion Objects

```scala
// class — a type blueprint; instantiate with new (or via apply())
class Counter(start: Int) {
  private var count = start
  def increment(): Unit = count += 1
  def value: Int = count
}
val c1 = new Counter(0)
val c2 = new Counter(10)  // independent instance

// object — a singleton: exactly one instance, created lazily on first access
// No new — it is already instantiated
object Config {
  val host = "localhost"
  val port = 5432
  def jdbcUrl = s"jdbc:postgresql://$host:$port/db"
}
Config.jdbcUrl  // access directly, no instantiation

// Companion object — an object with the same name as a class, in the same file
// Can access each other's private members
// Primary use: factory methods and constants related to the class
case class Order(id: Long, total: BigDecimal, status: OrderStatus)

object Order {
  val empty: Order = Order(0L, BigDecimal(0), OrderStatus.Pending)

  def fromCsv(line: String): Order = {
    val Array(id, total, status) = line.split(",")
    Order(id.toLong, BigDecimal(total), OrderStatus.valueOf(status))
  }
}

Order.fromCsv("42,99.99,Confirmed")  // factory — no new needed
Order.empty

// apply() makes new optional for any class
class Point(val x: Int, val y: Int)
object Point {
  def apply(x: Int, y: Int): Point = new Point(x, y)
}
val p = Point(3, 4)   // calls Point.apply — same as new Point(3, 4)
// case classes generate apply() automatically (that is why new is optional for them)
```

---

## Trait vs Abstract Class — Decision Guide

```scala
// Trait — interface with optional partial implementation
// Can be mixed in with `with`, enabling multiple inheritance
trait JsonSerializable { def toJson: String }
trait Auditable { def auditLog: List[String] }

class Order(...) extends JsonSerializable with Auditable {
  def toJson = s"""{"id": $id}"""
  def auditLog = List(s"order $id created")
}
// Can mix multiple traits — impossible with abstract classes

// Abstract class — base class with constructor parameters
// Use when the base type needs to receive values at construction time
abstract class Repository(val tableName: String) {
  def findById(id: Long): Option[?]
  protected val table: String = tableName
}

class OrderRepository extends Repository("orders") {
  def findById(id: Long): Option[Order] = ???
}
// Traits in Scala 2 cannot have constructor parameters (Scala 3 relaxes this)

// Decision rules:
//
//  Situation                                                   │ Use
//  ────────────────────────────────────────────────────────────┼─────────────────────
//  Behaviour reused in unrelated classes                       │ trait
//  Multiple behaviours composed into one class                 │ trait (with `with`)
//  No constructor parameters needed                            │ trait
//  Base class needs constructor parameters                     │ abstract class
//  Java clients will extend this type                          │ abstract class
//  Distributing a compiled library for others to subclass      │ abstract class
//  Modeling a domain with finite variants (ADT)                │ sealed trait
//  Instantiable, concrete domain type                          │ case class / class

// Classic pattern — sealed trait + case objects for ADTs:
sealed trait OrderStatus
case object Pending   extends OrderStatus
case object Confirmed extends OrderStatus
case object Shipped   extends OrderStatus
// compiler enforces exhaustive match — all cases must be handled
```

---

## Closures

```scala
// Basic closure — captures an immutable val
val multiplier = 3
val triple = (x: Int) => x * multiplier  // captures multiplier
triple(5)   // 15

// Closure over a var — the function sees updates to the variable
var factor = 3
val scale = (x: Int) => x * factor
scale(5)   // 15
factor = 10
scale(5)   // 50 — reflects the change; this is an impure closure

// Purity rule:
// Closure over val → pure (same as passing the val as a parameter)
// Closure over var → impure (hidden mutable state — avoid in production logic)

// Factory pattern — captures a fixed val, returns a specialized function
def makeAdder(n: Int): Int => Int = (x: Int) => x + n   // n is captured at call time
val addFive = makeAdder(5)
val addTen  = makeAdder(10)
addFive(3)  // 8
addTen(3)   // 13

// Practical use: inject configuration, close over a constant
val vatRate = BigDecimal("0.21")
val applyVAT: BigDecimal => BigDecimal = amount => amount * (1 + vatRate)
prices.map(applyVAT)  // vatRate captured once, applied to every price

// Any variable used inside a function that is NOT a parameter is a free variable.
// Free variables in a closure are its hidden inputs — make them explicit when possible.
```

---

## Tuples

```scala
// Creation — parentheses with comma-separated values
val pair: (Int, String)             = (1, "one")
val triple: (String, Int, Boolean)  = ("alice", 30, true)

// Element access — 1-indexed _N accessors (prefer destructuring)
pair._1    // 1
pair._2    // "one"

// Destructuring — the idiomatic way to unpack a tuple
val (num, word)         = pair
val (name, age, active) = triple

// Pattern matching on tuples
def label(t: (Int, String)): String = t match {
  case (n, s) if n > 0 => s"positive: $s"
  case (0, _)           => "zero"
  case (_, s)           => s"negative: $s"
}

// Returning multiple values from a function
def minMax(xs: List[Int]): (Int, Int) = (xs.min, xs.max)
val (lo, hi) = minMax(List(3, 1, 4, 1, 5, 9, 2))

// Map iteration — Map[K, V] is Iterable[(K, V)], each element is a tuple
Map("a" -> 1, "b" -> 2).map { case (key, value) => s"$key=$value" }
// Map("a" -> 1) is syntax sugar for Map(("a", 1)) — -> creates a tuple

// When to use tuple vs case class:
//
//  Tuple      → short-lived intermediate value, 2–3 fields, no reuse across functions
//  Case class → named fields, reused by multiple functions, > 3 fields, needs methods
//
// Rule: if you find yourself writing ._1, ._2 everywhere, extract a case class
case class Range(min: Int, max: Int)  // better than (Int, Int) when reused widely
```

---

## String Interpolation and Formatting

```scala
// s"" — simple substitution; calls .toString on each interpolated expression
val name = "Alice"
val age  = 30
s"Hello $name, you are $age years old"     // "Hello Alice, you are 30 years old"
s"Result: ${1 + 2}"                        // arbitrary expression inside ${}
s"User: ${user.name.trim.toUpperCase}"     // method chains inside ${}

// f"" — printf-style numeric formatting, type-checked at compile time
val price = 9.9
f"Price: $price%.2f"                       // "Price: 9.90"
f"$name%-10s | $age%3d"                   // left-pad name 10 chars, right-pad age 3

// raw"" — no escape sequence processing; use for regex patterns
raw"first\nsecond"                         // literal \n, not a newline
"first\\nsecond".r                         // raw"" is cleaner for regex

// Triple-quoted strings — multiline, no escaping needed
val query = """
  SELECT id, name
  FROM   users
  WHERE  active = true
""".stripMargin.trim

// .format() — Java-style; useful when the format string is a runtime value
val template = "Hello %s, you are %d years old"
template.format("Alice", 30)

// Prefer s"" for simple interpolation.
// Use f"" when precision formatting is required (currency, percentages).
// Use raw"" for regex patterns to avoid double-escaping backslashes.
```

---

## if / else as an Expression

`if / else` is an expression — it evaluates to a value. There is no ternary operator (`?:`) because `if / else` already serves that role.

```scala
// Returns a value — assign directly to a val
val label = if (amount > 1000) "large" else "small"   // String

// Both branches must return compatible types
val result = if (n > 0) n * 2 else 0                  // Int

// When branches have different types, Scala infers the nearest common supertype
val mixed = if (flag) 42 else "no"    // type: Any  (Int | String → Any)
// Avoid this — use a sealed trait to make the types explicit instead

// if with no else: implicit else branch returns Unit
val u = if (debug) println("debug")   // type: Unit, equivalent to:
val u = if (debug) println("debug") else ()

// Prefer val over conditional var:
val s = if (x > 0) "positive" else "non-positive"   // immutable, no null

// for / match are also expressions and return values by the same principle
val result = xs match {
  case Nil      => 0
  case h :: _ => h
}
```

---

## Type Inference — Let the Compiler Work

Annotate only at public API boundaries or when inference picks the wrong type.

```scala
// Infer everything the compiler can see
val n       = 42
val name    = "Alice"
val active  = true
val items   = List(1, 2, 3)
val lookup  = Map("a" -> 1, "b" -> 2)
val maybe   = Option("value")

// Inferred through transformations — no annotations needed
val doubled  = items.map(_ * 2)
val strings  = items.map(_.toString)
val filtered = items.filter(_ > 1)
val total    = items.foldLeft(0)(_ + _)

// Annotate public method return types — documents the contract
def processOrder(id: Long): Either[String, Order] = ???
def findUser(id: Long): Option[User] = ???

// Annotate when inference picks the wrong numeric type
val price = 0.0           // Double by default — OK
val exact = BigDecimal(0) // must be explicit, inference gives Int for literals
val id    = 0L            // L suffix forces Long without annotation

// Annotate when creating an empty collection with a specific type
val empty = List.empty[String]  // or List[String]()
val buf   = Map.empty[String, Int]
```

---

## Values, Variables, Lazy Evaluation

```scala
// Always prefer val — immutable binding
val name = "Alice"
val count = 42

// var only at system boundaries (parsing CLI args, mutable accumulator in tight loop)
var mutableCounter = 0      // smell: ask why mutation is needed

// lazy val — evaluated once, on first access
lazy val schema = expensiveSchemaLoad()   // computed only when first used

// Type aliases for intention-revealing names
type UserId    = Long
type Email     = String
type OrderId   = java.util.UUID
```

---

## Case Classes — Immutable Domain Models

```scala
case class User(id: Long, name: String, email: String, role: Role)
case class Order(id: UUID, userId: Long, items: List[OrderItem], status: OrderStatus)
case class OrderItem(productId: Long, quantity: Int, unitPrice: BigDecimal)

// Update as you copy, don't mutate
val original = User(1L, "Alice", "alice@example.com", Role.Admin)
val renamed  = original.copy(name = "Alicia")  // original unchanged

// Structural equality — no boilerplate
case class Point(x: Int, y: Int)
Point(1, 2) == Point(1, 2)  // true
Point(1, 2).hashCode == Point(1, 2).hashCode  // true

// toString for free
Point(3, 4).toString  // "Point(3,4)"
```

**Value classes — zero-cost type safety** (prevent mixing IDs):
```scala
case class UserId(value: Long)    extends AnyVal
case class OrderId(value: Long)   extends AnyVal
case class AmountUSD(value: Double) extends AnyVal

// Compiler prevents: findOrder(orderId, userId) → compile error!
def findOrder(userId: UserId, orderId: OrderId): Option[Order]
// At runtime, UserId is just a Long — no boxing overhead
```

---

## Sealed Traits — Algebraic Data Types

The compiler enforces exhaustive matching — missing a case is a compile error.

```scala
sealed trait OrderStatus
case object Pending   extends OrderStatus
case object Confirmed extends OrderStatus
case object Shipped   extends OrderStatus
case object Cancelled extends OrderStatus

sealed trait PaymentResult
case class  PaymentSuccess(transactionId: String, amount: BigDecimal) extends PaymentResult
case class  PaymentFailure(code: String, reason: String)              extends PaymentResult
case object PaymentPending                                             extends PaymentResult

// Exhaustive match — compiler warns if a case is missing
def describe(status: OrderStatus): String = status match {
  case Pending   => "Awaiting confirmation"
  case Confirmed => "Confirmed"
  case Shipped   => "On its way"
  case Cancelled => "Cancelled"
  // No default needed — sealed + exhaustive = compiler guarantees coverage
}
```

---

## Pattern Matching

```scala
// Type and structure
def processResult(result: PaymentResult): String = result match {
  case PaymentSuccess(txId, amount) => s"Paid $$amount (tx: $txId)"
  case PaymentFailure(code, reason) => s"Failed [$code]: $reason"
  case PaymentPending               => "Awaiting payment"
}

// Guard conditions
def classify(n: Int): String = n match {
  case x if x < 0      => "negative"
  case 0                => "zero"
  case x if x % 2 == 0 => s"positive even: $x"
  case x                => s"positive odd: $x"
}

// Nested destructuring
case class Address(city: String, country: String)
case class Person(name: String, address: Address)

def greet(p: Person): String = p match {
  case Person(name, Address(city, "ES")) => s"Hola $name, de $city"
  case Person(name, Address(city, _))   => s"Hello $name, from $city"
}

// List patterns (recursive algorithms)
def sum(xs: List[Int]): Int = xs match {
  case Nil          => 0
  case head :: tail => head + sum(tail)
}

// Tuple patterns
(1, "hello", true) match {
  case (n, s, true)  => s"$n: $s (enabled)"
  case (n, s, false) => s"$n: $s (disabled)"
}
```

---

## Error Handling — No Exceptions in Pure Code

Pure functions don't throw. Return a type that encodes the possibility of failure.

```scala
// Option — value may be absent, no info about why
def findUser(id: Long): Option[User] = users.get(id)

val greeting = findUser(42)
  .map(u => s"Hello, ${u.name}")
  .getOrElse("User not found")

// Either — failure with a reason. Convention: Left = error, Right = success
def validateEmail(raw: String): Either[String, String] =
  if (raw.contains("@")) Right(raw.trim.toLowerCase)
  else Left(s"Invalid email: '$raw'")

def validateAge(age: Int): Either[String, Int] =
  if (age >= 18 && age <= 120) Right(age)
  else Left(s"Age must be 18–120, got $age")

// Chain validations with for comprehension — stops at first Left
def createProfile(rawEmail: String, age: Int): Either[String, Profile] =
  for {
    email        <- validateEmail(rawEmail)
    validatedAge <- validateAge(age)
  } yield Profile(email, validatedAge)

// Try — wraps exceptions from Java libraries
import scala.util.{Try, Success, Failure}

def parseConfig(raw: String): Try[Config] = Try(JsonParser.parse(raw))

parseConfig(input) match {
  case Success(config)    => useConfig(config)
  case Failure(exception) => log.error("Config parse failed", exception)
}

// Convert between types
optionValue.toRight("error when None")   // Option → Either
eitherValue.toOption                      // Either → Option (loses Left info)
Try(riskyOp()).toEither                   // Try → Either
```

**When to use which** (see `references/fp-patterns.md` for full matrix):
- `Option`: absent value, no error info needed
- `Either`: one thing can fail, want the error
- `Try`: wrapping Java exceptions at the boundary
- `Validated`: accumulate ALL validation errors (see fp-patterns.md)

---

## By-Name Parameters — Lazy Arguments

```scala
// Call-by-value: argument evaluated before function is called
def logV(msg: String): Unit = if (isDebug) println(msg)
logV(expensiveMsg())  // expensiveMsg() runs even if !isDebug ← waste

// Call-by-name: argument evaluated each time it's referenced inside the function
def logN(msg: => String): Unit = if (isDebug) println(msg)
logN(expensiveMsg())  // expensiveMsg() only runs if isDebug ← correct

// By-name enables custom control structures
def withRetry[A](maxAttempts: Int)(thunk: => A): A = {
  var remaining = maxAttempts
  var result: Option[A] = None
  while (result.isEmpty && remaining > 0) {
    result = Try(thunk).toOption
    remaining -= 1
  }
  result.getOrElse(throw new RuntimeException(s"Failed after $maxAttempts attempts"))
}

// Reads like a built-in control structure
withRetry(3) {
  callExternalApi()
}

// Timer utility
def timer[A](block: => A): (A, Double) = {
  val start  = System.nanoTime
  val result = block
  val ms     = (System.nanoTime - start) / 1e6
  (result, ms)
}

val (result, ms) = timer(runHeavyQuery())
```

---

## Multiple Parameter Groups and Currying

```scala
// Multiple groups
def sum(a: Int)(b: Int)(c: Int): Int = a + b + c
sum(1)(2)(3)  // 6

// Partial application — fix some parameters
def withTimeout[A](seconds: Int)(thunk: => A): A = { /* ... */ thunk }
val withTenSeconds = withTimeout(10) _    // partially applied: Int baked in
withTenSeconds { callApi() }

// Type inference benefit: the type of `b` is inferred from `a`
def transform[A, B](list: List[A])(f: A => B): List[B] = list.map(f)
transform(List(1, 2, 3))(n => n * 2)  // B inferred as Int

// Implicit + explicit parameters in separate groups
def query[A](sql: String)(implicit conn: Connection): A = { /* ... */ ??? }

// Currying for specialization
def multiply(a: Int)(b: Int): Int = a * b
val double = multiply(2) _   // partially applied
val triple = multiply(3) _

List(1, 2, 3).map(double)  // List(2, 4, 6)
List(1, 2, 3).map(triple)  // List(3, 6, 9)
```

---

## Higher-Order Functions and Collections

Collections are immutable. Transform them — never mutate in place.

```scala
val orders: List[Order] = loadOrders()

orders.map(_.total)                             // transform each
orders.filter(_.status == Confirmed)            // keep matching
orders.flatMap(_.items)                         // transform and flatten
orders.foldLeft(BigDecimal(0))(_ + _.total)     // reduce
orders.groupBy(_.status)                        // partition into Map
orders.sortBy(_.createdAt)                      // sort
orders.partition(_.total > 100)                 // split into (yes, no)
orders.exists(_.status == Cancelled)            // any match?
orders.forall(_.amount > 0)                     // all match?
orders.count(_.region == "EU")                  // count matching
orders.take(10)                                 // first N
orders.zipWithIndex.map { case (o, i) => s"${i+1}. ${o.id}" }

// collect: filter + map in one (partial function)
val revenues: List[BigDecimal] = results.collect {
  case PaymentSuccess(_, amount) => amount
}

// Compose as a pipeline — reads like prose
val report = orders
  .filter(_.status == Shipped)
  .sortBy(_.createdAt)
  .take(10)
  .map(o => s"${o.id}: ${o.total}")
  .mkString("\n")
```

---

## For Comprehensions — Monadic Composition

For comprehensions are syntactic sugar for `flatMap`/`map`/`filter`. They work on any type that has those methods — `Option`, `Either`, `Try`, `Future`, `List`, `IO`, and custom types.

```scala
// Without for (callback pyramid)
findUser(userId)
  .flatMap(user => findOrder(orderId)
    .flatMap(order => validatePayment(user, order)
      .map(payment => processPayment(payment))))

// With for (reads sequentially — same semantics)
val result: Option[Receipt] =
  for {
    user    <- findUser(userId)    // if None, whole expression is None
    order   <- findOrder(orderId)
    payment <- validatePayment(user, order)
  } yield processPayment(payment)

// Either — stops at first Left
val processed: Either[AppError, Receipt] =
  for {
    user    <- findUser(userId).toRight(UserNotFound(userId))
    order   <- findOrder(orderId).toRight(OrderNotFound(orderId))
    receipt <- chargeUser(user, order)
  } yield receipt

// Filter in for comprehension
val evens: List[Int] =
  for {
    x <- List(1, 2, 3, 4, 5, 6)
    if x % 2 == 0           // filter step
  } yield x * x             // List(4, 16, 36)

// Future — sequential async
val response: Future[ApiResponse] =
  for {
    user    <- userService.find(userId)
    account <- accountService.find(user.accountId)
    resp    <- paymentService.charge(account, amount)
  } yield resp
```

---

## Tail Recursion

Annotate with `@tailrec` to get compile-time guarantee of no stack overflow.

```scala
import scala.annotation.tailrec

// Tail-recursive sum — compiler converts to a loop (no stack growth)
def sum(xs: List[Int]): Int = {
  @tailrec
  def loop(remaining: List[Int], acc: Int): Int = remaining match {
    case Nil          => acc
    case head :: tail => loop(tail, acc + head)  // tail call: nothing after recursive call
  }
  loop(xs, 0)
}

// NOT tail-recursive — stack grows with list length
def sumNaive(xs: List[Int]): Int = xs match {
  case Nil          => 0
  case head :: tail => head + sumNaive(tail)  // `+` happens AFTER the recursive call
}

// Pattern: accumulator parameter = tail-recursive
@tailrec
def reverse[A](xs: List[A], acc: List[A] = Nil): List[A] = xs match {
  case Nil          => acc
  case head :: tail => reverse(tail, head :: acc)
}

@tailrec
def find[A](xs: List[A], predicate: A => Boolean): Option[A] = xs match {
  case Nil                       => None
  case head :: _ if predicate(head) => Some(head)
  case _ :: tail                 => find(tail, predicate)
}
```

---

## Type Classes — Ad-hoc Polymorphism

```scala
// Define the type class
trait JsonEncoder[A] {
  def encode(value: A): String
}

// Instances
given JsonEncoder[User] with
  def encode(u: User): String = s"""{"id":${u.id},"name":"${u.name}"}"""

given JsonEncoder[Order] with
  def encode(o: Order): String = s"""{"id":"${o.id}","total":${o.total}}"""

// Context bound syntax — concise
def toJson[A: JsonEncoder](value: A): String =
  summon[JsonEncoder[A]].encode(value)  // Scala 3

// Or: implicitly[JsonEncoder[A]].encode(value)  // Scala 2

toJson(user)   // uses JsonEncoder[User]
toJson(order)  // uses JsonEncoder[Order]

// Compose instances
given [A: JsonEncoder]: JsonEncoder[List[A]] with
  def encode(xs: List[A]) = xs.map(toJson).mkString("[", ",", "]")

toJson(List(user1, user2))  // uses JsonEncoder[List[User]] → uses JsonEncoder[User]
```

**Variance in type classes:**
```scala
trait Container[+A]   // covariant: Container[Cat] <: Container[Animal]
trait Consumer[-A]    // contravariant: Consumer[Animal] can be used as Consumer[Cat]
// Rule: covariant for producers (read-only), contravariant for consumers (write-only)
// See references/type-system.md for full variance reference
```

---

## Clean Code in Scala

### 16 Naming Rules

```scala
// 1. Intention-revealing: name says WHAT and WHY
val d = 0                         // bad: what is d?
val elapsedTimeInDays = 0         // good

// 2. No disinformation: don't imply the wrong thing
val accountList = Set("acc-1")    // bad: it's a Set, not a List!
val accountIds  = Set("acc-1")    // good

// 3. Meaningful distinctions: not o1, o2, a, b
def copy(a1: String, a2: String): Unit  // bad
def copy(source: String, dest: String): Unit  // good

// 4. Pronounceable: if you can't say it, you can't discuss it
val genymdhms = Instant.now()         // bad
val generationTimestamp = Instant.now()  // good

// 5. Searchable: no magic numbers or single-letter variables
items.length * 5                      // bad: what is 5?
val WORK_DAYS_PER_WEEK = 5
items.length * WORK_DAYS_PER_WEEK     // good

// 6. No encodings: no type prefix, no m_ member prefix
val strEmail = "a@b.com"   // bad
val m_userId = 42L         // bad
val email    = "a@b.com"   // good
val userId   = 42L         // good

// 7. No mental mapping: don't make the reader decode abbreviations
val r = getRow(id)         // bad: r is not self-documenting
val orderRow = getRow(id)  // good

// 8. Class names: nouns (User, Order, OrderProcessor, Repository)
class Process { }          // bad: verb
class OrderProcessor { }   // good: noun

// 9. Method names: verbs (findUser, validateEmail, calculateTotal)
def user(id: Long): Option[User] = ???  // bad: noun
def findUser(id: Long): Option[User] = ???  // good: verb

// 10. No cuteness: clear over clever
def whack(items: List[Item]): Unit = items.foreach(delete)    // bad
def deleteItems(items: List[Item]): Unit = items.foreach(delete)  // good

// 11. One word per concept: pick get OR fetch OR retrieve, use it everywhere
def fetchUser(id: Long): Option[User]    // UserService
def getUser(id: Long): Option[User]      // UserController
// ↑ inconsistency — pick one and use it across the whole codebase

// 12. No puns: don't reuse a word for a different purpose
// If "add" already means "add two numbers", don't use "add" for "append to list"
// Use "append" or "insert" instead

// 13. Solution domain names: use algorithm/pattern names for developers
// Factory, Visitor, Strategy, Queue, Tree — these are legitimate names

// 14. Problem domain names: use business terminology when no CS term fits
// "amortizationSchedule", "proratedFee", "fiscalPeriod"

// 15. Meaningful context: fields without context are ambiguous
val street = "..."        // bad: what kind of address?
val orderStreet = "..."   // good: clearly part of an order address
// Better: extract Address case class — context from the enclosing type

// 16. No gratuitous context: don't repeat the class name in its fields
class User { val userName = "Alice" }  // bad: redundant "User" in "userName"
class User { val name = "Alice" }      // good
```

### Function Argument Rules

```scala
// Niladic (0 args): ideal — no coupling, easy to test, no temporal dependency
def now(): Instant = Instant.now()

// Monadic (1 arg): clean and clear
def parseEmail(raw: String): Either[String, Email] = ???

// Dyadic (2 args): acceptable — but consider if an object is hiding
def writeFile(path: Path, content: String): IO[Unit] = ???

// Triadic (3 args): reconsider — almost always signals a missing abstraction
// Usually two of the three params form a cohesive concept
def createOrder(customer: Customer, payment: PaymentMethod, address: Address): Order
// (vs) def createOrder(customerId: Long, customerName: String, customerEmail: String,
//                     paymentMethod: String, street: String, city: String): Order

// Flag (Boolean) arguments: ALWAYS wrong — they announce the function does two things
def save(user: User, sendWelcomeEmail: Boolean): IO[Unit]  // bad: two functions hiding here
// Fix: two clearly named functions
def save(user: User): IO[Unit] = ???
def saveAndWelcome(user: User): IO[Unit] = save(user) *> sendWelcome(user)

// Output arguments: mutating a passed parameter instead of returning a value
def addItem(item: Item, order: mutable.ListBuffer[Item]): Unit = order += item  // bad
def addItem(item: Item, order: List[Item]): List[Item] = item :: order          // good

// Command-Query Separation: do something OR return something, never both
def saveAndReturn(order: Order): Order = { db.save(order); order }  // bad: two things
def save(order: Order): IO[Unit] = IO(db.save(order))               // good: command only
// The caller already has the Order — return it at the call site if needed
```

### Comment Taxonomy

```scala
// ─── GOOD COMMENTS ───────────────────────────────────────────────────────────

// Legal: copyright at the top of the file (one line)
// Copyright (c) 2024 Example Corp. All rights reserved.

// Explanation of intent — WHY, not WHAT (the code already says WHAT)
// Using bubble sort: profiling shows this input is nearly sorted 99.9% of the time;
// bubble sort outperforms quicksort for nearly-sorted inputs below 1000 elements.
items.sortWith(bubbleSort)

// Warning — consequence the reader must know before using this
// NOT thread-safe: the underlying connection pool library is not reentrant.
// Wrap in synchronized or use one instance per thread.
def callLegacyApi(req: Request): Response = ???

// TODO with context — acknowledge known debt, track it
// TODO: replace with ValidatedNel after ValidationService refactor (JIRA-2481)
def validateAge(age: Int): Either[String, Int] = ???

// ─── BAD COMMENTS ────────────────────────────────────────────────────────────

// Redundant: the code already says this — delete it
val total = items.map(_.price).sum  // sum the prices of all items

// Journal: use git log instead — delete it
// 2024-01-10 Alice: added validation
// 2024-03-15 Bob: fixed null handling

// Commented-out code: use git to retrieve it if needed — delete it
// def legacyProcess(order: Order): Unit = { database.insert(order) }

// Mandated javadoc with no content: delete it
/**
 * Gets the user.
 * @param id the id
 * @return the user
 */
def getUser(id: Long): Option[User] = ???

// Noise: says nothing — delete it
// Default constructor
case class Order(id: Long, total: BigDecimal)
// Increment counter
counter += 1
```

### Formatting — Newspaper Metaphor

- The **headline** (class/object name) tells you what this is about
- The **opening paragraph** (public methods at top) gives the high-level story
- **Details expand** as you read down (private helpers at bottom)
- **Caller always appears above callee** — the reader never needs to scroll up

```scala
// Good: public interface at top, helpers below
object OrderService {
  def processOrder(order: Order): IO[Either[AppError, Receipt]] =    // public — high level
    for {
      validated <- IO.fromEither(validateOrder(order))
      receipt   <- chargeAndRecord(validated)
    } yield Right(receipt)

  private def validateOrder(order: Order): Either[AppError, Order] = ???  // helper
  private def chargeAndRecord(order: Order): IO[Receipt] = ???            // helper
}

// Vertical openness: blank line between logically unrelated lines
val userId  = extractUserId(token)
val user    = loadUser(userId)

val products  = loadProducts(user.preferences)
val featured  = selectFeatured(products, user.region)
// The blank line signals a new concept — parseable without comments
```

### Kent Beck's 4 Rules of Simple Design

In priority order:

```
1. Runs all tests
   Testability forces good design. Untestable code is a smell — it has hidden
   dependencies or violates SRP. If you can't test it, redesign it.

2. No duplication (DRY)
   Extract when you see the same shape twice. But resist the urge to extract
   at first glance — wait for the third occurrence. Premature abstraction is
   as bad as duplication.

3. Expresses the programmer's intent
   Self-documenting code: clear names, small functions, the right abstractions.

4. Minimizes the number of classes and methods
   No speculative abstractions. No "we might need this later."
```

### Data/Object Anti-Symmetry and Law of Demeter

```scala
// ─── Anti-Symmetry ───────────────────────────────────────────────────────────

// Data structure: exposes fields, no behavior — easy to add new functions to
case class Point(x: Double, y: Double)  // just data
def distance(a: Point, b: Point): Double = ???  // add functions freely
// Adding a new Point type requires updating all functions

// Object: hides data, exposes behavior — easy to add new types
trait Shape { def area: Double }
case class Circle(radius: Double) extends Shape { def area = math.Pi * radius * radius }
case class Square(side: Double)   extends Shape { def area = side * side }
// Adding Triangle: add one class, zero changes to area callers
// Adding perimeter: must touch all classes

// In FP (Scala style): prefer data structures + functions for stable types + evolving ops
// In OOP (polymorphism): prefer trait hierarchy for stable ops + evolving types
// Scala lets you use both: case class hierarchies with companion object functions

// ─── Law of Demeter — Talk to your immediate friends ─────────────────────────

// Bad: train wreck — you're reaching deep into implementation
val discount = user.getAccount().getMembership().getTier().getDiscount()
// This couples your code to the full internal chain: User → Account → Membership → Tier

// Good: ask the object, let it figure out its internals
val discount = user.membershipDiscount()  // User knows its own discount
// If Membership changes internally, only User.membershipDiscount() needs updating
```

---

## TDD in Scala — Red, Green, Refactor

```scala
// Step 1: Write the failing test (RED)
class DiscountSpec extends AnyFunSpec with Matchers {
  describe("applyDiscount") {
    it("gives VIP users 10% off") {
      val order = Order(id = UUID.randomUUID(), total = BigDecimal(100))
      applyDiscount(VIPUser, order).total shouldBe BigDecimal(90)
    }
    it("gives regular users no discount") {
      val order = Order(id = UUID.randomUUID(), total = BigDecimal(100))
      applyDiscount(RegularUser, order).total shouldBe BigDecimal(100)
    }
    it("never increases the order total") {
      val order = Order(id = UUID.randomUUID(), total = BigDecimal(200))
      applyDiscount(VIPUser, order).total should be <= order.total
    }
  }
}

// Step 2: Write minimum code to pass (GREEN)
def applyDiscount(user: UserType, order: Order): Order = user match {
  case VIPUser     => order.copy(total = order.total * 0.9)
  case RegularUser => order
}

// Step 3: Refactor with confidence (tests catch regressions)
```

**Property-based testing with ScalaCheck:**
```scala
import org.scalacheck.{Gen, Prop}
import org.scalacheck.Prop.forAll

class DiscountProperties extends Properties("applyDiscount") {
  val orderGen  = Gen.posNum[Double].map(n => Order(UUID.randomUUID(), BigDecimal(n)))
  val userGen   = Gen.oneOf(VIPUser, RegularUser)

  property("discount never increases the order total") = forAll(orderGen, userGen) {
    (order, user) => applyDiscount(user, order).total <= order.total
  }

  property("discount result is always non-negative") = forAll(orderGen, userGen) {
    (order, user) => applyDiscount(user, order).total >= BigDecimal(0)
  }
}
```

**Test sealed traits exhaustively — the compiler helps:**
```scala
sealed trait Shape
case class Circle(radius: Double)          extends Shape
case class Rectangle(w: Double, h: Double) extends Shape

def area(shape: Shape): Double = shape match {
  case Circle(r)        => Math.PI * r * r
  case Rectangle(w, h)  => w * h
  // If you add Triangle but forget to handle it here → compile warning
}

class ShapeSpec extends AnyFlatSpec with Matchers {
  "area" should "compute circle area" in {
    area(Circle(1.0)) shouldBe (Math.PI +- 0.001)
  }
  it should "compute rectangle area" in {
    area(Rectangle(3.0, 4.0)) shouldBe 12.0
  }
}
```

**Push IO to the edge — pure core is trivially testable:**
```scala
// Pure core: test without mocks
def buildUserReport(users: List[User]): String =
  users.sortBy(_.name).map(u => s"${u.name}: ${u.email}").mkString("\n")

// Impure edge: test the integration separately with real/stub dependencies
class UserReportIntegrationSpec extends AnyFunSpec {
  it("fetches users and builds report") {
    val stubRepo = new UserRepository { def findAll(): IO[List[User]] = IO(testUsers) }
    val result   = UserReportService(stubRepo).generate().unsafeRunSync()
    result should include("Alice")
  }
}
```

---

## Futures and Concurrency

```scala
import scala.concurrent.{Future, ExecutionContext}
import scala.concurrent.ExecutionContext.Implicits.global

// Sequential (each step waits for previous)
val result: Future[Receipt] =
  for {
    user    <- userService.find(userId)
    account <- accountService.find(user.accountId)  // waits for user
    receipt <- paymentService.charge(account, amount)
  } yield receipt

// Parallel (start both futures BEFORE for comprehension)
val userFuture  = userService.find(userId)    // started immediately
val orderFuture = orderService.find(orderId)  // started immediately

for {
  user  <- userFuture   // wait for both
  order <- orderFuture
} yield process(user, order)

// Error handling
future
  .map(result => Right(result))
  .recover { case e => Left(e.getMessage) }
  .map {
    case Right(r) => use(r)
    case Left(e)  => log.warn(s"Failed: $e")
  }
```

---

## Reference Files

- `references/fp-patterns.md` — Functor, Monad, Applicative, Kleisli, IO + Resource, Validated, State, monad laws, recursion patterns, function composition, hidden inputs diagnostic
- `references/type-system.md` — Variance, type bounds, higher-kinded types, phantom types, opaque types (Scala 3), union/intersection types
- `references/collections.md` — Collection hierarchy, performance table, groupMap/groupMapReduce, builder patterns, specialized collections
- `references/domain-modeling.md` — Case classes, sealed trait ADTs, smart constructors, opaque types, refinement types
- `references/concurrency-futures.md` — Futures, ExecutionContext, async patterns, Cats Effect IO
- `references/testing.md` — Full TDD reference: F.I.R.S.T. principles, BUILD-OPERATE-CHECK, one assert per test, ScalaTest DSL, ScalaCheck generators, mocking, TestContainers, IO testing
- `references/clean-architecture.md` — SOLID full Scala examples, component cohesion/coupling, Clean Architecture layers, screaming architecture, Humble Object, frameworks as details
- `references/code-smells.md` — 40+ named smells (naming, function, comment, data, class, general, test) with Scala examples and fixes
