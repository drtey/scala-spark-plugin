# Clean Architecture and Design Principles for Scala

---

## The Three Programming Paradigms

| Paradigm | Removes | Enables | Scala expression |
|---------|---------|---------|-----------------|
| Structured | Unrestricted jumps (goto) | Provable control flow | `if/else`, `for`, `while` — no breaks |
| Object-Oriented | Unrestricted function pointers | Safe polymorphism | `sealed trait` + pattern matching |
| Functional | Variable assignment | Referential transparency | `val`, pure functions, immutable collections |

> "All three are needed. Use FP for core business logic. Use OOP-style polymorphism for plugin points. Use structured control flow everywhere."

**Scala is all three simultaneously:**
```scala
// Structured: disciplined control flow
def findFirst(items: List[Int], pred: Int => Boolean): Option[Int] =
  items match {
    case Nil             => None           // structured base case
    case h :: _ if pred(h) => Some(h)     // structured early return via pattern
    case _ :: rest       => findFirst(rest, pred)
  }

// Functional: pure, immutable core
def applyDiscount(userType: UserType, total: BigDecimal) = userType match {
  case VIP     => total * BigDecimal("0.9")
  case Regular => total
}

// OOP-style: polymorphism via sealed trait
sealed trait PaymentMethod
case class CreditCard(last4: String) extends PaymentMethod
case class BankTransfer(iban: String) extends PaymentMethod
def feeFor(method: PaymentMethod) = method match {
  case CreditCard(_)  => BigDecimal("0.029")
  case BankTransfer(_) => BigDecimal("0.001")
}
```

---

## SOLID Principles — Full Definitions with Scala

### SRP — Single Responsibility Principle

> "A module has one and only one **reason to change**."

A "reason to change" means a specific user/stakeholder whose requests would modify the module. If three different stakeholders would request changes to the same class, that class has three responsibilities.

```scala
// Bad: Payroll serves employees (pay calc), finance (reporting), and treasury (payments)
class Payroll {
  def calculatePay(employee: Employee): BigDecimal    = ???  // HR / Payroll team
  def reportHours(employee: Employee): String         = ???  // Finance team
  def depositFunds(employee: Employee, amount: BigDecimal): IO[Unit] = ???  // Treasury
}
// Any change by Finance (report format) forces recompile of everything in this class

// Good: one actor per class
class PayCalculator {
  def calculate(employee: Employee): BigDecimal =
    employee.hourlyRate * employee.hoursWorked
}

class PayrollReporter {
  def report(employees: List[Employee]): String =
    employees.map(e => s"${e.name}: ${e.hoursWorked}h").mkString("\n")
}

class FundDepositor {
  def deposit(employee: Employee, amount: BigDecimal): IO[Unit] =
    IO(bankAPI.transfer(employee.bankAccount, amount))
}
```

### OCP — Open-Closed Principle

> "Open for extension, closed for modification."

Add new behavior by adding new code; do not modify existing code. The key mechanism: depend on abstractions (traits), implement concretions.

```scala
// Bad: adding a new payment type requires modifying this function
def calculateFee(paymentType: String, amount: BigDecimal): BigDecimal = paymentType match {
  case "credit_card"   => amount * BigDecimal("0.029")
  case "bank_transfer" => amount * BigDecimal("0.001")
  // Adding "crypto"? MUST modify this code.
}

// Good with type classes (Scala idiom) — closed for modification, open for extension
trait FeeCalculator[P] {
  def fee(method: P, amount: BigDecimal): BigDecimal
}

case class CreditCard(last4: String)
case class BankTransfer(iban: String)
case class CryptoPayment(address: String)  // NEW type — zero changes to existing code

given FeeCalculator[CreditCard] with {
  def fee(m: CreditCard, amount: BigDecimal) = amount * BigDecimal("0.029")
}
given FeeCalculator[BankTransfer] with {
  def fee(m: BankTransfer, amount: BigDecimal) = amount * BigDecimal("0.001")
}
given FeeCalculator[CryptoPayment] with {  // NEW instance — no old code touched
  def fee(m: CryptoPayment, amount: BigDecimal) = amount * BigDecimal("0.005")
}

def calculateFee[P: FeeCalculator](method: P, amount: BigDecimal) =
  summon[FeeCalculator[P]].fee(method, amount)
```

### LSP — Liskov Substitution Principle

> "Subtypes must be substitutable for their supertypes without breaking correctness."

Classic violation: `Square extends Rectangle`. Square enforces `width == height`, but code that uses `Rectangle` assumes it can set width and height independently. Square breaks that assumption.

```scala
// Classic violation in OOP (translated to Scala)
abstract class Rectangle {
  var width: Double
  var height: Double
  def area = width * height
}

class Square(side: Double) extends Rectangle {
  width = side; height = side
  override def width_=(w: Double) = { width = w; height = w }  // breaks contract!
}

val r: Rectangle = new Square(5)
r.width  = 10
r.height = 20
r.area   // Expected: 200. Actual: 400. LSP violated.

// Good: honest ADT — Rectangle and Square are siblings, not parent-child
sealed trait Shape { def area: Double }
case class Rectangle(width: Double, height: Double) extends Shape { def area = width * height }
case class Square(side: Double) extends Shape { def area = side * side }

// Both are substitutable for Shape:
def totalArea(shapes: List[Shape]) = shapes.map(_.area).sum
totalArea(List(Rectangle(10, 20), Square(5)))  // correct for both
```

### ISP — Interface Segregation Principle

> "Don't depend on things you don't use."

```scala
// Bad: DataProcessor has 6 methods; ETLPipeline uses only 3
trait DataProcessor {
  def read(source: String): Data
  def validate(data: Data): Either[String, Data]
  def transform(data: Data): Data
  def persist(data: Data): IO[Unit]
  def report(data: Data): String      // ETLPipeline doesn't need this
  def archive(data: Data): IO[Unit]   // ETLPipeline doesn't need this
}

class ETLPipeline(processor: DataProcessor) {
  def run(source: String): IO[Unit] = ???
  // compiles against the full DataProcessor trait — depends on report and archive
}

// Good: segregated traits — each client depends only on what it uses
trait DataReader    { def read(source: String): Data }
trait DataValidator { def validate(data: Data): Either[String, Data] }
trait DataTransformer { def transform(data: Data): Data }
trait DataPersister  { def persist(data: Data): IO[Unit] }

class ETLPipeline(
  reader:      DataReader,
  validator:   DataValidator,
  transformer: DataTransformer,
  persister:   DataPersister
) {
  def run(source: String): IO[Unit] = ???
}
// No dependency on report or archive
```

### DIP — Dependency Inversion Principle

> "High-level modules should not depend on low-level modules. Both should depend on abstractions."
> "Source code dependencies should point toward abstractions."

**Stable** = abstract traits, interfaces, pure functions — rarely changes.
**Volatile** = concrete implementations (databases, HTTP clients, email APIs) — changes often.

Depend on the stable. Keep the volatile at the edges.

```scala
// Bad: high-level business logic depends on volatile low-level detail
class OrderService {
  private val db    = new PostgresDatabase()  // volatile — specific driver
  private val email = new SmtpEmailClient()   // volatile — specific protocol

  def processOrder(order: Order): Unit = {
    db.save(order)
    email.send(order.userId, "Your order is confirmed")
  }
}

// Good: business logic depends on stable abstractions
trait OrderRepository { def save(order: Order): IO[Unit] }
trait NotificationService { def notify(userId: UserId, message: String): IO[Unit] }

class OrderService(repo: OrderRepository, notifier: NotificationService) {
  def processOrder(order: Order): IO[Unit] =
    for {
      _ <- repo.save(order)
      _ <- notifier.notify(order.userId, "Your order is confirmed")
    } yield ()
}

// Volatile implementations live at the edges (infrastructure layer)
class PostgresOrderRepository extends OrderRepository { def save(o: Order) = IO(db.insert(o)) }
class SmtpNotificationService extends NotificationService { def notify(uid, msg) = IO(smtp.send(uid, msg)) }

// Main (the edge) wires everything together
object Main extends IOApp.Simple {
  def run = {
    val repo     = new PostgresOrderRepository()
    val notifier = new SmtpNotificationService()
    new OrderService(repo, notifier).processOrder(testOrder)
  }
}
```

---

## Component Cohesion Principles

These answer: *what belongs together in a component (JAR / module / package)?*

**REP — Reuse/Release Equivalence Principle**
> The granule of reuse is the granule of release.
- If someone reuses your code, it must be versioned and released as a unit
- Don't force users to download unrelated code in the same package

**CCP — Common Closure Principle**
> Gather together the classes that change at the same times and for the same reasons.
- If two classes always change together when a feature changes, they should be in the same component
- Minimizes the number of components that change for any single requirement

**CRP — Common Reuse Principle**
> Don't force users of a component to depend on things they don't need.
- If you import a component, you depend on ALL of it — breaking changes anywhere affect you
- Classes not reused together should not be packaged together

**The tension:** REP and CCP say "group more together"; CRP says "group less together". Balance by considering the stage of your project.

---

## Component Coupling Principles

These answer: *in which direction should component dependencies point?*

**ADP — Acyclic Dependencies Principle**
> The dependency structure of components must be a directed acyclic graph (DAG).
- Component A → B → C is OK. A → B → A is a cycle — BAD.
- Cycles prevent independent compilation, deployment, and testing.

```scala
// BAD: cycle
// package users depends on package auth
// package auth depends on package users
// Solution: extract the shared concept to a third package both can depend on
// users → shared-domain ← auth
```

**SDP — Stable Dependencies Principle**
> Depend in the direction of stability.
- Stability = (fan-in) / (fan-in + fan-out) where fan-in = "depended on by"
- A component depended on by many and depending on few is stable (hard to change)
- Don't let a stable component depend on a volatile one

**SAP — Stable Abstractions Principle**
> A component should be as abstract as it is stable.
- Stable components should be abstract (traits, interfaces) — extensible without modifying
- Volatile components should be concrete (implementations) — easy to change
- Zone of pain: stable + concrete (can't change; can't extend)
- Zone of uselessness: volatile + abstract (abstract but nobody depends on it)

```scala
// Good component structure (SDP + SAP applied):
// domain/         — stable, abstract (traits, case classes, pure functions)
// services/       — medium stability, depends on domain abstractions
// infrastructure/ — volatile, concrete (DB, HTTP, email implementations)
// main/           — most volatile, wires everything, depends on all

// Dependency direction: main → infrastructure → services → domain
// domain depends on NOTHING (innermost, most stable)
```

---

## Clean Architecture Layers

From innermost (most stable) to outermost (most volatile):

```
┌─────────────────────────────────────────────┐
│  Frameworks & Drivers                        │  HTTP, database, UI, Kafka
│  ┌─────────────────────────────────────┐     │
│  │  Interface Adapters                 │     │  Controllers, Gateways, Presenters
│  │  ┌─────────────────────────────┐   │     │
│  │  │  Application Business Rules │   │     │  Use Cases, Application Logic
│  │  │  ┌───────────────────────┐ │   │     │
│  │  │  │ Enterprise Rules      │ │   │     │  Domain Entities, Pure Logic
│  │  │  └───────────────────────┘ │   │     │
│  │  └─────────────────────────────┘   │     │
│  └─────────────────────────────────────┘     │
└─────────────────────────────────────────────┘

THE DEPENDENCY RULE: all source code dependencies point INWARD.
No inner layer can name or import anything from an outer layer.
```

**Translated to Scala packages:**
```scala
// domain/           (innermost) — pure case classes, sealed traits, pure functions
// usecases/         — orchestrate domain logic; use trait abstractions for I/O
// adapters/         — convert between use-case types and framework types
// infrastructure/   — DB implementations, HTTP handlers, event producers
// main/             — wires everything; the only place that depends on all layers
```

**Full example:**
```scala
// DOMAIN (innermost — zero dependencies)
object OrderDomain {
  case class Order(id: OrderId, items: List[Item], total: BigDecimal, status: OrderStatus)

  def validateOrder(order: Order): Either[ValidationError, Order] =
    Either.cond(order.items.nonEmpty && order.total > 0, order, EmptyOrderError)

  def applyDiscount(userType: UserType, order: Order): Order =
    userType match {
      case VIP => order.copy(total = order.total * BigDecimal("0.9"))
      case _   => order
    }
}

// USE CASE — depends on domain traits/types, NOT on frameworks
trait OrderRepository { def save(order: Order): IO[Unit] }

class CreateOrderUseCase(repo: OrderRepository) {
  def execute(items: List[Item], userType: UserType): IO[Either[AppError, OrderId]] =
    OrderDomain.validateOrder(Order(newId(), items, items.map(_.price).sum, Pending))
      .map(o => OrderDomain.applyDiscount(userType, o))
      .fold(
        err => IO.pure(Left(err)),
        order => repo.save(order).map(_ => Right(order.id))
      )
}

// ADAPTER — converts HTTP types to use-case types
class OrderController(useCase: CreateOrderUseCase) {
  def create(req: HttpRequest): IO[HttpResponse] =
    useCase.execute(parseItems(req), parseUserType(req)).map {
      case Left(err)  => HttpResponse(400, err.message)
      case Right(id)  => HttpResponse(201, id.toString)
    }
}

// INFRASTRUCTURE — concrete implementations
class PostgresOrderRepository extends OrderRepository {
  def save(order: Order) = IO(sql"INSERT INTO orders VALUES (${order.id})".execute())
}

// MAIN — wires it all (only place with full knowledge of all layers)
object Main extends IOApp.Simple {
  def run = {
    val repo       = PostgresOrderRepository()
    val useCase    = CreateOrderUseCase(repo)
    val controller = OrderController(useCase)
    HttpServer.bind(8080, controller).use(_ => IO.never)
  }
}
```

---

## Screaming Architecture

> "The architecture of a system should scream its intent."

The source directory structure must immediately communicate what the *system does*, not what framework it uses.

```
BAD (screams "Rails", "Spring", "web application"):
  app/
    controllers/
    models/
    views/
    services/

GOOD (screams "Order Management System"):
  domain/
    orders/
    customers/
    payments/
  application/
    usecases/CreateOrder.scala
    usecases/CancelOrder.scala
    usecases/RefundOrder.scala
  infrastructure/
    db/PostgresOrderRepository.scala
    http/OrderController.scala
    events/KafkaOrderPublisher.scala
```

---

## Humble Object Pattern

Split non-testable code (framework glue) from testable logic. The humble object is thin — it just delegates.

```scala
// Non-testable shell (HTTP framework, DB transaction)
class OrderController(handler: OrderHandler) {
  def create(req: HttpRequest): IO[HttpResponse] =
    handler.handle(parseRequest(req)).map(toResponse)
  // "handle" and parsing are separately testable
}

// Testable parsing (pure)
def parseRequest(req: HttpRequest): Either[ParseError, CreateOrderRequest] =
  for {
    items    <- req.body.items.asRight
    userType <- UserType.fromString(req.body.userType)
  } yield CreateOrderRequest(items, userType)

// Testable response conversion (pure)
def toResponse(result: Either[AppError, OrderId]): HttpResponse = result match {
  case Left(err) => HttpResponse(400, err.message)
  case Right(id) => HttpResponse(201, id.toString)
}

// Both pure functions are tested directly — no HTTP needed in tests
test("parseRequest rejects missing user type") {
  val req = HttpRequest(body = """{"items": []}""")
  parseRequest(req) shouldBe Left(MissingUserTypeError)
}
```

---

## The Details Are Details

> "The database is a detail. The web is a detail. Frameworks are details."

Business logic must never know:
- Whether the database is Postgres, MongoDB, or in-memory
- Whether the transport is REST, gRPC, GraphQL, or CLI
- Which version of the HTTP library you're using

```scala
// BAD: business logic knows about HTTP
class DiscountService {
  def apply(req: HttpRequest): HttpResponse = {
    val amount = req.body("amount").toDouble
    val discounted = amount * 0.9
    HttpResponse(200, discounted.toString)
  }
}

// GOOD: business logic is framework-agnostic
def applyDiscount(amount: BigDecimal): BigDecimal = amount * BigDecimal("0.9")

// HTTP adapter calls it:
class DiscountController {
  def apply(req: HttpRequest): HttpResponse = {
    val amount     = BigDecimal(req.body("amount"))
    val discounted = applyDiscount(amount)
    HttpResponse(200, discounted.toString)
  }
}

// CLI adapter also calls it — zero changes to the business logic:
object DiscountCLI extends App {
  val amount     = BigDecimal(args(0))
  val discounted = applyDiscount(amount)
  println(discounted)
}
```

If you can test your business logic without starting an HTTP server or connecting to a database, your architecture is clean.

---

## Applying Architecture to Spark

Spark is a framework. It belongs at the outer layers.

```scala
// BAD: business logic knows about Spark
def processOrders(df: DataFrame): DataFrame = {
  df.filter($"status" === "active")
    .withColumn("discounted", $"amount" * lit(0.9))
}
// Impossible to test without SparkSession; Catalyst can't optimize custom business rules

// GOOD: separate business logic from Spark
// Inner layer — pure Scala, no Spark
def isActive(order: Order): Boolean = order.status == "active" && order.amount > 0
def applyDiscount(order: Order): Order = order.copy(amount = order.amount * 0.9)

// Outer layer — Spark wraps the pure functions
def processOrders(ds: Dataset[Order]): Dataset[Order] =
  ds.filter(isActive).map(applyDiscount)

// Tests for business logic (no Spark):
isActive(Order(1L, "active", 100.0)) shouldBe true
applyDiscount(Order(1L, "active", 100.0)).amount shouldBe 90.0

// Tests for Spark transformation (local session):
processOrders(testDataset).count() shouldBe 1
```
