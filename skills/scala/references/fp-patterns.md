# Functional Programming Patterns in Scala

## Pure Functions

A function is pure if:
1. It always returns the same output for the same input (referential transparency)
2. It has no side effects (no I/O, no mutation, no exceptions)

```scala
// Pure
def add(a: Int, b: Int): Int = a + b
def toUpperCase(s: String): String = s.toUpperCase
def formatUser(u: User): String = s"${u.name} <${u.email}>"

// Impure (has side effects)
def saveUser(u: User): Unit = database.insert(u)  // I/O
def getTimestamp(): Long = System.currentTimeMillis()  // non-deterministic
var total = 0
def addToTotal(n: Int): Unit = { total += n }  // mutates external state

// Strategy: push impure code to the edges; keep the core pure
// Pure core:
def buildInsertQuery(u: User): String = s"INSERT INTO users VALUES (${u.id}, '${u.name}')"
// Impure edge:
def saveUser(u: User): IO[Unit] = IO(database.execute(buildInsertQuery(u)))
```

## Functor — Mapping over a Context

A Functor is any type `F[A]` that supports `map: (A => B) => F[B]`.

```scala
// Option is a Functor
Some(42).map(_ * 2)           // Some(84)
(None: Option[Int]).map(_ * 2) // None

// List is a Functor
List(1, 2, 3).map(_ * 2)      // List(2, 4, 6)

// Either is a Functor (maps over Right)
Right(10).map(_ + 5)           // Right(15)
Left("error").map(_ + 5)       // Left("error") — unchanged

// Write generic code with any Functor (using Cats)
import cats.Functor
import cats.syntax.functor._

def double[F[_]: Functor](fa: F[Int]): F[Int] = fa.map(_ * 2)
double(Option(5))       // Some(10)
double(List(1, 2, 3))  // List(2, 4, 6)
```

## Monad — Sequential Computation with Context

A Monad is a Functor with `flatMap: (A => F[B]) => F[B]`. Enables chaining operations where each step may introduce a context (failure, async, etc.).

```scala
// Option monad — chain computations that may be absent
def findUser(id: Long): Option[User]       = users.get(id)
def findAccount(user: User): Option[Account] = accounts.get(user.accountId)
def getBalance(acc: Account): Option[BigDecimal] = Some(acc.balance)

// Without monad — nested matches
findUser(42) match {
  case Some(user) => findAccount(user) match {
    case Some(acc) => getBalance(acc)
    case None      => None
  }
  case None => None
}

// With flatMap (monadic chaining)
findUser(42).flatMap(findAccount).flatMap(getBalance)

// With for comprehension (syntactic sugar for flatMap + map)
val balance: Option[BigDecimal] =
  for {
    user    <- findUser(42)
    account <- findAccount(user)
    balance <- getBalance(account)
  } yield balance
```

## Monoid — Combining Values

A Monoid provides `combine: (A, A) => A` and an `empty: A` identity element.

```scala
// String Monoid
"Hello" + ", " + "World"  // combine = +, empty = ""

// Int (addition) Monoid: combine = +, empty = 0
List(1, 2, 3, 4).foldLeft(0)(_ + _)  // = 10

// Use case: reduce any List[A] if A is a Monoid
def combineAll[A](items: List[A], empty: A, combine: (A, A) => A): A =
  items.foldLeft(empty)(combine)

// With Cats
import cats.Monoid
import cats.syntax.monoid._

implicit val orderMonoid: Monoid[OrderSummary] = new Monoid[OrderSummary] {
  def empty = OrderSummary(0, BigDecimal(0))
  def combine(a: OrderSummary, b: OrderSummary) =
    OrderSummary(a.count + b.count, a.total + b.total)
}

orders.map(toSummary).combineAll  // reduces all summaries to one
```

## IO Monad — Referentially Transparent I/O

The IO monad wraps side effects so they become values — lazy, composable, and testable.

```scala
// Simple IO implementation (concept; use Cats Effect in production)
case class IO[A](unsafeRun: () => A) {
  def map[B](f: A => B): IO[B]         = IO(() => f(unsafeRun()))
  def flatMap[B](f: A => IO[B]): IO[B] = IO(() => f(unsafeRun()).unsafeRun())
}

object IO {
  def apply[A](thunk: => A): IO[A] = new IO(() => thunk)
  def pure[A](a: A): IO[A]        = IO(() => a)
}

// Usage: describe the program as a pure value
val readLine: IO[String]          = IO(scala.io.StdIn.readLine())
def printLine(s: String): IO[Unit] = IO(println(s))

val program: IO[Unit] =
  for {
    _    <- printLine("Enter your name:")
    name <- readLine
    _    <- printLine(s"Hello, $name!")
  } yield ()

// Nothing runs until you call unsafeRun at the very edge of the program
program.unsafeRun()  // main entry point only

// With Cats Effect (production-ready)
import cats.effect.{IO, IOApp}

object Main extends IOApp.Simple {
  def run: IO[Unit] =
    for {
      _    <- IO.println("Enter your name:")
      name <- IO(scala.io.StdIn.readLine())
      _    <- IO.println(s"Hello, $name!")
    } yield ()
}
```

## Error Accumulation with Validated

`Either` short-circuits on first error. `Validated` accumulates all errors.

```scala
import cats.data.{Validated, ValidatedNel, NonEmptyList}
import cats.syntax.validated._
import cats.syntax.apply._

type ValidationResult[A] = ValidatedNel[String, A]

def validateName(name: String): ValidationResult[String] =
  if (name.nonEmpty) name.validNel
  else "Name cannot be empty".invalidNel

def validateEmail(email: String): ValidationResult[String] =
  if (email.contains("@")) email.validNel
  else "Invalid email format".invalidNel

def validateAge(age: Int): ValidationResult[Int] =
  if (age >= 18) age.validNel
  else "Must be 18 or older".invalidNel

// Accumulates ALL errors (not just the first)
def createUser(name: String, email: String, age: Int): ValidationResult[User] =
  (validateName(name), validateEmail(email), validateAge(age))
    .mapN(User.apply)

createUser("", "invalid", 15)
// Invalid(NonEmptyList(
//   "Name cannot be empty",
//   "Invalid email format",
//   "Must be 18 or older"
// ))
```

## State Monad — Threading State Purely

```scala
import cats.data.State

// State[S, A] = S => (S, A)
// Thread state through computations without mutation

type GameState = Map[String, Int]

def addScore(player: String, points: Int): State[GameState, Unit] =
  State.modify(state => state.updated(player, state.getOrElse(player, 0) + points))

def getScore(player: String): State[GameState, Int] =
  State.inspect(_.getOrElse(player, 0))

val game: State[GameState, String] =
  for {
    _      <- addScore("Alice", 10)
    _      <- addScore("Bob", 15)
    _      <- addScore("Alice", 5)
    aScore <- getScore("Alice")
    bScore <- getScore("Bob")
  } yield s"Alice: $aScore, Bob: $bScore"

val (finalState, result) = game.run(Map.empty).value
// result = "Alice: 15, Bob: 15"
// finalState = Map("Alice" -> 15, "Bob" -> 15)
```

## Error Handling Strategy Matrix

| Situation | Use |
|-----------|-----|
| Value may be absent, no error info | `Option[A]` |
| Computation may fail, want error info | `Either[Error, A]` |
| Wrapping Java exceptions | `Try[A]` then `.toEither` |
| Validate multiple fields, accumulate ALL errors | `ValidatedNel[E, A]` |
| Sequential steps where any can fail | `Either` in for comprehension |
| Side effects that may fail | `IO[A]` (Cats Effect) |

```scala
// Option — absence, no reason
def findConfig(key: String) = sys.env.get(key)

// Either — failure with reason
def parsePort(s: String): Either[String, Int] =
  s.toIntOption.filter(p => p > 0 && p < 65536).toRight(s"Invalid port: $s")

// Try — wraps Java exceptions
import scala.util.Try
def parseJson(raw: String) = Try(ujson.read(raw)).toEither.left.map(_.getMessage)

// ValidatedNel — accumulate all errors, not just first
// (see Error Accumulation section)

// for comprehension — sequential: stops at first Left
val result =
  for {
    port <- parsePort(sys.env.getOrElse("PORT", "0"))
    host <- sys.env.get("HOST").toRight("HOST is required")
  } yield Config(host, port)
```

---

## Applicative — Parallel Independent Effects

Monad sequences effects (each step depends on the previous). Applicative combines independent effects.

```scala
import cats.syntax.apply._
import cats.syntax.validated._
import cats.data.ValidatedNel

// Monad (Either) — sequential, stops on first error
val monadResult =
  for {
    name  <- validateName(raw.name)    // if this fails, nothing else runs
    email <- validateEmail(raw.email)
    age   <- validateAge(raw.age)
  } yield User(name, email, age)

// Applicative (ValidatedNel) — parallel, accumulates all errors
val applicativeResult =
  (validateName(raw.name), validateEmail(raw.email), validateAge(raw.age))
    .mapN(User.apply)

// Rule: use Applicative when effects are independent (validation, parallel config parsing)
//       use Monad when each step depends on the previous result

// Example: form validation returns all errors at once
def validateForm(raw: RawForm): ValidatedNel[String, User] =
  (
    validateName(raw.name),
    validateEmail(raw.email),
    validateAge(raw.age)
  ).mapN(User.apply)

validateForm(RawForm("", "not-an-email", -5))
// Invalid(NonEmptyList("Name cannot be empty", "Invalid email", "Age must be >= 0"))
```

---

## Kleisli — Composing Effectful Functions

Kleisli wraps `A => F[B]` and allows composing functions that return effects.

```scala
import cats.data.Kleisli
import cats.implicits._

// Without Kleisli — hard to compose A => Either[E, B] functions
def parseId(raw: String): Either[String, Long]    = raw.toLongOption.toRight(s"Not a number: $raw")
def findUser(id: Long): Either[String, User]      = userRepo.get(id).toRight(s"User $id not found")
def checkActive(user: User): Either[String, User] = Either.cond(user.active, user, "User is inactive")

// Manual composition
def resolveUser(raw: String): Either[String, User] =
  parseId(raw).flatMap(findUser).flatMap(checkActive)

// With Kleisli — composable, reusable pipeline
val parseIdK   = Kleisli(parseId)
val findUserK  = Kleisli(findUser)
val checkActiveK = Kleisli(checkActive)

val pipeline = parseIdK andThen findUserK andThen checkActiveK

pipeline.run("42")       // Either[String, User]
pipeline.run("abc")      // Left("Not a number: abc")
pipeline.run("99")       // Left("User 99 not found")

// Local modification — run a step with transformed input
val enriched = pipeline.local[String](_.trim)
enriched.run("  42  ")   // works after trimming
```

---

## IO + Resource — Safe Resource Management

`Resource[IO, A]` guarantees acquisition and release, even on error.

```scala
import cats.effect.{IO, Resource}
import java.sql.{Connection, DriverManager}

// Define a managed resource
val dbConnection: Resource[IO, Connection] =
  Resource.make(
    IO(DriverManager.getConnection(jdbcUrl, user, pass))  // acquire
  )(conn =>
    IO(conn.close())  // always runs, even if body throws
  )

// Use — connection is released after the block, guaranteed
val result = dbConnection.use { conn =>
  IO(conn.prepareStatement("SELECT * FROM users").executeQuery())
}

// Compose resources
val resources: Resource[IO, (Connection, KafkaProducer)] =
  (dbConnection, kafkaProducer).tupled

// Use multiple resources
resources.use { (conn, producer) =>
  for {
    rows    <- IO(conn.query("SELECT id FROM orders"))
    _       <- producer.send(rows.map(toEvent))
  } yield ()
}

// Resource.fromAutoCloseable — for Java AutoCloseable
val fileResource = Resource.fromAutoCloseable(IO(new FileInputStream("data.csv")))
```

---

## Referential Transparency — The Key Principle

A function is referentially transparent if you can replace any call with its return value without changing behavior.

```scala
// Referentially transparent — can substitute freely
val x = add(1, 2)   // x = 3
add(1, 2) + add(1, 2) == x + x  // always true

// NOT referentially transparent
var counter = 0
def increment(): Int = { counter += 1; counter }
val a = increment()  // a = 1, counter = 1
val b = increment()  // b = 2, counter = 2
// a + b ≠ increment() + increment() — depends on call order

// The goal: maximize the referentially transparent core
// Push all increment-style state to State monad, IO monad, or the very edges
```

---

## Hidden Inputs — The Purity Diagnostic

```scala
// Clue 1: no input parameters → function must be reading from somewhere (hidden input)
def getCurrentUser(): User = sessionStore.get("user")   // reads global session
def getTodayRevenue(): BigDecimal = db.query("SELECT ...")  // reads DB

// Clue 2: returns Unit → function must be writing somewhere (hidden output = side effect)
def logOrder(order: Order): Unit = println(order)           // writes stdout
def saveMetric(name: String, value: Long): Unit = influx.write(name, value)

// Free variable diagnostic:
// Any variable used inside the function that is NOT a parameter is a free variable (hidden input)
val TAX_RATE = 0.21  // free variable — captured from outer scope

def calculateTax(amount: BigDecimal): BigDecimal = amount * TAX_RATE  // impure: reads free var
// Problem: tests must know the outer scope value; changing TAX_RATE breaks tests silently

// Fix: make the hidden input explicit
def calculateTax(amount: BigDecimal, taxRate: BigDecimal): BigDecimal = amount * taxRate

// Reward: trivially testable, no shared state
calculateTax(BigDecimal(100), BigDecimal("0.21")) shouldBe BigDecimal(21)
```

**Diagnostic checklist for any function you're about to test:**
1. Does it have zero parameters? → where does it get its data? make it explicit.
2. Does it return Unit? → what is it writing? wrap in IO or return the value.
3. Does it access a `var`, a singleton, or an external system directly? → inject it.

---

## Recursion Patterns

```scala
import scala.annotation.tailrec

// Pattern: identify base case (when to stop) + recursive case (how to reduce)

// ── Factorial ──────────────────────────────────────────────────────────────
// Not tail-recursive: * happens AFTER the recursive call, so each frame is kept
def factorial(n: Int): BigInt = n match {
  case 0 => 1
  case n => n * factorial(n - 1)  // n stays on stack waiting for recursive result
}

// Tail-recursive: @tailrec verifies at compile time — stack-safe
@tailrec
def factorial(n: Int, acc: BigInt = 1): BigInt = n match {
  case 0 => acc
  case n => factorial(n - 1, n * acc)  // tail position: no work after recursive call
}

// ── List sum ───────────────────────────────────────────────────────────────
@tailrec
def sum(xs: List[Int], acc: Int = 0): Int = xs match {
  case Nil       => acc
  case h :: rest => sum(rest, acc + h)
}

// ── Fibonacci ──────────────────────────────────────────────────────────────
// Naive (exponential — teaching only, never use in production)
def fib(n: Int): BigInt = n match {
  case 0 => 0
  case 1 => 1
  case n => fib(n - 1) + fib(n - 2)  // not tail-recursive, exponential calls
}

// Linear tail-recursive (production)
@tailrec
def fib(n: Int, a: BigInt = 0, b: BigInt = 1): BigInt = n match {
  case 0 => a
  case n => fib(n - 1, b, a + b)  // shift window: a=b, b=a+b
}

// ── Find ───────────────────────────────────────────────────────────────────
@tailrec
def find[A](xs: List[A], p: A => Boolean): Option[A] = xs match {
  case Nil              => None
  case h :: _ if p(h)  => Some(h)
  case _ :: t           => find(t, p)
}

// Rule: if you need an accumulator → add it as a default parameter
// Rule: if the recursive call is the LAST expression in its branch → it's tail-recursive
// Rule: annotate with @tailrec — the compiler rejects non-tail-recursive usages
```

---

## Function Composition

```scala
val double  = (x: Int) => x * 2
val addOne  = (x: Int) => x + 1

// compose: right to left — f compose g means f(g(x))
val addThenDouble = double compose addOne  // double(addOne(x))
addThenDouble(5)  // double(6) = 12

// andThen: left to right — f andThen g means g(f(x))
val addThenDouble2 = addOne andThen double  // double(addOne(x))
addThenDouble2(5)  // 12 — same result, reads left-to-right (more natural)

// Compose a data pipeline (reads in execution order with andThen)
val normalizeText = (s: String) => s.trim.toLowerCase
val tokenize      = (s: String) => s.split("\\W+").toList.filter(_.nonEmpty)
val removeShort   = (tokens: List[String]) => tokens.filter(_.length >= 3)

val preprocess: String => List[String] =
  normalizeText andThen tokenize andThen removeShort

preprocess("  Hello World, this is a TEST!  ")
// List("hello", "world", "this", "test")

// Named intermediate stages are self-documenting
// Each stage is independently testable as a pure function
```

---

## Monad Laws

The three monad laws ensure that flatMap and pure compose predictably. Violating them produces surprising behavior.

```scala
// Law 1 — Left identity: wrapping a value and immediately flatMapping = just calling f
// pure(a).flatMap(f) == f(a)
Option(42).flatMap(x => Option(x * 2)) == Option(42 * 2)  // true
// Intuition: pure() adds no effect; flatMap immediately unwraps it

// Law 2 — Right identity: flatMapping with pure = identity
// m.flatMap(pure) == m
Option(42).flatMap(x => Option(x)) == Option(42)  // true
// Intuition: pure() adds no effect; flatMap unwraps cleanly back to original

// Law 3 — Associativity: nesting of flatMap doesn't matter
// m.flatMap(f).flatMap(g) == m.flatMap(x => f(x).flatMap(g))
val lhs = Option(10).flatMap(x => Option(x + 1)).flatMap(y => Option(y * 2))
val rhs = Option(10).flatMap(x => Option(x + 1).flatMap(y => Option(y * 2)))
lhs == rhs  // true: Option(22) == Option(22)

// These laws guarantee that for comprehensions can be freely refactored.
// A type that claims to be a Monad but violates these laws will produce bugs
// that are impossible to reason about from the types alone.
```

---

## Monad from First Principles

```scala
// A Wrapper is a Monad — build it from scratch to see why
case class Wrapper[A](value: A) {
  def map[B](f: A => B): Wrapper[B]         = Wrapper(f(value))
  def flatMap[B](f: A => Wrapper[B]): Wrapper[B] = f(value)
}

// Wrapper works in for comprehensions because it has map + flatMap
val result =
  for {
    a <- Wrapper(10)
    b <- Wrapper(a * 2)   // b depends on a — sequential, not parallel
    c <- Wrapper(a + b)
  } yield c
// result: Wrapper(30)

// Now look at the standard library types through this lens:
// Option   = Wrapper where value may be absent (None short-circuits the chain)
// List     = Wrapper with multiple values (flatMap = cross-product)
// Either   = Wrapper with two possible types (Left short-circuits)
// Try      = Wrapper where evaluation may throw (Failure short-circuits)
// IO       = Wrapper where evaluation is deferred (nothing runs until run())
// Future   = Wrapper where evaluation is async

// ALL of these share the same shape: map + flatMap.
// That shared shape is what "Monad" names.
```
