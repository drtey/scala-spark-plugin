# Concurrency and Futures in Scala

## The Core Rule: Never Block Inside a Future

Blocking a thread inside a Future or IO starves the ExecutionContext. Use async chaining instead.

```scala
// Wrong — blocks an EC thread while waiting for another Future
Future {
  val result = Await.result(otherFuture, 5.seconds)
  process(result)
}

// Right — chain with flatMap or for comprehension
val result = otherFuture.flatMap(value => Future(process(value)))

for {
  data      <- fetchData()
  processed <- Future(process(data))
} yield processed
```

---

## ExecutionContext: Which One to Use

| Situation | ExecutionContext |
|-----------|----------------|
| CPU-bound computation | `ExecutionContext.global` (sized to CPU count) |
| Blocking I/O (JDBC, filesystem) | Dedicated fixed or cached thread pool |
| Fire-and-forget side effects | Dedicated single-thread EC |

```scala
// Default — use for CPU-bound work
implicit val ec: ExecutionContext = ExecutionContext.global

// Dedicated pool for blocking I/O — do not block on the global EC
val ioEc: ExecutionContext =
  ExecutionContext.fromExecutorService(
    java.util.concurrent.Executors.newFixedThreadPool(32)
  )

def readFromDatabase(id: Long): Future[Order] =
  Future(db.blockingFindById(id))(ioEc)  // explicit EC, not the implicit global
```

Rule: if a `Future` body calls a blocking API (`Thread.sleep`, JDBC, file I/O), it must run on a dedicated IO ExecutionContext. Blocking on the global EC causes thread starvation under concurrent load.

---

## Sequential vs Parallel Composition

Futures start executing when they are created, not when they are `flatMap`ped. Creating them inside a `for` comprehension makes them sequential.

```scala
// Sequential — each step waits for the previous (correct when b depends on a)
val result: Future[C] = for {
  a <- fetchA()
  b <- fetchB(a)  // depends on a
  c <- combine(a, b)
} yield c

// Parallel — start both before the for comprehension
val fa = fetchA()   // starts immediately
val fb = fetchB()   // starts immediately, concurrently with fa
val result: Future[(A, B)] = for {
  a <- fa
  b <- fb
} yield (a, b)

// Wrong — looks parallel but is sequential (fb starts only after fa completes)
val result = for {
  a <- fetchA()  // fa starts
  b <- fetchB()  // fb starts only after fa finishes
} yield (a, b)
```

---

## Error Handling in Futures

Do not use `Await.result` or `.get` in production paths. Handle failures inside the Future chain.

```scala
// recover — transform a failure into a fallback value
fetchOrder(id).recover {
  case _: NotFoundException => Order.empty
  case e: TimeoutException  => throw e  // re-throw non-recoverable errors
}

// recoverWith — fall back to another Future
fetchFromCache(id).recoverWith {
  case _: CacheMiss => fetchFromDatabase(id)
}

// Convert to Either to make errors explicit
val result: Future[Either[String, Order]] = fetchOrder(id)
  .map(Right(_))
  .recover { case e => Left(e.getMessage) }
```

---

## Cats Effect IO: The Correct Alternative

Prefer `IO` over `Future` in new code. `IO` is referentially transparent, lazily evaluated, and does not require implicit `ExecutionContext` threading through your signatures.

```scala
import cats.effect.IO

// IO is lazy — describes an effect, does not run it
val fetchOrder: IO[Order] = IO(db.findById(id))

// Run only at the program edge (main or test)
fetchOrder.unsafeRunSync()

// Parallel composition
val result: IO[(User, Orders)] = IO.both(
  IO(fetchUser(id)),
  IO(fetchOrders(id))
)

// Error handling
fetchOrder.attempt.map {
  case Right(order) => order
  case Left(e)      => Order.empty
}

// Guaranteed resource release
Resource
  .make(IO(db.startTransaction()))(tx => IO(tx.rollbackIfOpen()))
  .use(tx => IO(tx.execute(query)))
```

Wrapping legacy `Future`-based code: use `IO.fromFuture(IO(legacyFutureCall()))`.

---

## Shared Mutable State

Do not share mutable `var`s between concurrent Futures. Use atomic operations or `Ref` (Cats Effect) instead.

```scala
// Wrong — data race: multiple threads increment the same var
var counter = 0
(1 to 1000).foreach(_ => Future { counter += 1 })

// Right — atomic reference (Java interop)
val counter = new java.util.concurrent.atomic.AtomicInteger(0)
(1 to 1000).foreach(_ => Future { counter.incrementAndGet() })

// Right — Cats Effect Ref (pure, referentially transparent)
import cats.effect.{IO, Ref}
import cats.syntax.traverse._

val program = for {
  counter <- Ref[IO].of(0)
  _       <- (1 to 1000).toList.traverse(_ => counter.update(_ + 1))
  result  <- counter.get
} yield result
```

Rule: if two concurrent operations can both write to the same location, that location must be protected by an atomic operation or a `Ref`. A `val` pointing to a mutable collection is not safe.
