# Code Smells Reference

Each smell is a symptom of a design problem. The cure is usually: rename, extract, remove, or separate.

---

## Naming Smells

**Disinformation** — name implies the wrong thing
```scala
// Smell: using "List" for a non-List collection
val accountList = Set("acc-1", "acc-2")   // it's a Set, not a List!
// Fix:
val accountIds = Set("acc-1", "acc-2")
```

**Non-pronounceable name** — can't say it aloud, can't discuss it in code review
```scala
// Smell:
val crtnTmsp = Instant.now()
// Fix:
val creationTimestamp = Instant.now()
```

**Non-searchable name** — magic numbers, single-letter variables
```scala
// Smell:
val total = items.length * 5   // what is 5?
// Fix:
val WORK_DAYS_PER_WEEK = 5
val total = items.length * WORK_DAYS_PER_WEEK
```

**Encoding in name** — type prefixes, `m_` fields, Hungarian notation
```scala
// Smell:
val strEmail = "alice@example.com"
val m_userId = 42L
// Fix:
val email  = "alice@example.com"
val userId = 42L
```

**Mental mapping** — reader must decode abbreviations
```scala
// Smell:
val r = getRowFromDb(id)
// Fix:
val orderRow = getOrderFromDatabase(id)
```

**Verb for class name / noun for method name**
```scala
// Smell:
class Process { ... }           // class should be noun
def user(id: Long): Option[User] = ???  // method should be verb
// Fix:
class OrderProcessor { ... }
def findUser(id: Long): Option[User] = ???
```

**Cute name** — clever over clear
```scala
// Smell:
def whack(items: List[Item]): Unit = items.foreach(delete)
// Fix:
def deleteItems(items: List[Item]): Unit = items.foreach(delete)
```

**Multiple words for same concept** — inconsistency
```scala
// Smell: same operation named differently in different places
def fetchUser(id: Long): Option[User]   // UserService
def retrieveUser(id: Long): Option[User]  // UserRepository
def getUser(id: Long): Option[User]       // UserController
// Fix: pick one word (get / find / fetch) and use it everywhere
```

**Wrong level of abstraction in name** — exposes implementation detail
```scala
// Smell:
def getItemsFromMySQLOrdersTable(): List[Item]  // reveals storage technology
// Fix:
def findOrderItems(orderId: OrderId): List[Item]
```

---

## Function Smells

**Flag argument** — Boolean parameter means the function does two things
```scala
// Smell:
def save(user: User, sendWelcomeEmail: Boolean): IO[Unit]
// Fix: two clearly-named functions
def save(user: User): IO[Unit]
def saveAndWelcome(user: User): IO[Unit] = save(user) *> sendWelcome(user)
```

**Too many arguments** — signals missing abstraction
```scala
// Smell:
def createOrder(customerId: Long, customerName: String, customerEmail: String,
                itemIds: List[Long], paymentMethod: String, address: String): Order
// Fix: introduce parameter objects
case class Customer(id: Long, name: String, email: String)
case class ShippingAddress(street: String, city: String, country: String)
def createOrder(customer: Customer, items: List[ItemId],
                payment: PaymentMethod, address: ShippingAddress): Order
```

**Output argument** — function mutates a parameter instead of returning a value
```scala
// Smell:
def addItem(item: Item, order: mutable.ListBuffer[Item]): Unit = order += item
// Fix:
def addItem(item: Item, order: List[Item]): List[Item] = item :: order
```

**Command-Query violation** — function does something AND returns a value
```scala
// Smell: returns the old value AND mutates state
def popAndReturn(stack: mutable.Stack[Int]): Int = stack.pop()
// Fix: separate
def peek(stack: List[Int]): Option[Int] = stack.headOption
def pop(stack: List[Int]): List[Int]    = stack.tail
```

**Feature envy** — method uses more of another class than its own
```scala
// Smell: applyDiscount is envious of Membership
case class User(id: Long, membership: Membership)
case class Membership(tier: Tier, discount: Double, expiresAt: Instant)

def applyDiscount(user: User, amount: BigDecimal): BigDecimal = {
  if (user.membership.tier == Gold && user.membership.expiresAt.isAfter(Instant.now()))
    amount * (1 - user.membership.discount)
  else amount
}
// Fix: move the logic to where it belongs
case class Membership(tier: Tier, discount: Double, expiresAt: Instant) {
  def isActive: Boolean = expiresAt.isAfter(Instant.now())
  def applyTo(amount: BigDecimal): BigDecimal =
    if (tier == Gold && isActive) amount * (1 - discount) else amount
}
def applyDiscount(user: User, amount: BigDecimal) = user.membership.applyTo(amount)
```

**Dead function** — function is never called anywhere
```scala
// Smell: legacyProcess is never referenced
def legacyProcess(order: Order): Unit = { /* old code */ }
// Fix: DELETE IT. It's in git history if ever needed.
```

**Too many levels of abstraction in one function** — mixes high-level intent with low-level detail
```scala
// Smell: processOrder handles everything at all levels
def processOrder(order: Order): IO[Unit] = {
  val validated = if (order.items.nonEmpty && order.total > 0) Right(order) else Left("invalid")
  validated match {
    case Left(err) => IO(logger.error(s"Validation failed: $err"))
    case Right(o) =>
      IO {
        val conn = DriverManager.getConnection(jdbcUrl)
        val stmt = conn.prepareStatement("INSERT INTO orders ...")
        stmt.setLong(1, o.id.value)
        stmt.execute()
        conn.close()
      }
  }
}

// Fix: each level of abstraction gets its own function
def processOrder(order: Order): IO[Unit] =
  for {
    validated <- IO.fromEither(validateOrder(order))
    _         <- saveOrder(validated)
  } yield ()

def validateOrder(order: Order): Either[String, Order] = ???
def saveOrder(order: Order): IO[Unit] = ???
```

---

## Comment Smells

**Redundant comment** — repeats what the code already says
```scala
// Smell:
val total = items.map(_.price).sum  // sum the prices of all items
// Fix: delete the comment. The code is self-explanatory.
```

**Misleading comment** — inaccurate description
```scala
// Smell:
// Returns the first inactive user
def findUser(active: Boolean): Option[User] = users.find(_.active == active)
// Fix: fix the comment or the code
// Returns a user matching the given active status
```

**Mandated comment** — required by policy but carries no information
```scala
// Smell: forced javadoc with nothing useful
/**
 * Gets the user.
 * @param id the id
 * @return the user
 */
def getUser(id: Long): Option[User]
// Fix: a good name makes the comment unnecessary. Only document non-obvious contracts.
```

**Journal comment** — changelog in the code
```scala
// Smell:
// 2023-01-10 Alice: added validation
// 2023-03-15 Bob: fixed null handling
// 2024-01-05 Carol: refactored for new API
def validateUser(user: User): Either[String, User] = ???
// Fix: DELETE. git log --follow -p shows the entire history.
```

**Commented-out code**
```scala
// Smell:
// def legacyProcess(order: Order): Unit = {
//   database.insert(order)
//   emailService.send(order.userId)
// }
// Fix: DELETE. It's in git. Nobody reading this code knows whether to restore or ignore it.
```

**Noise comment** — says nothing
```scala
// Smell:
// Default constructor
case class Order(id: Long, total: BigDecimal)

// Increment counter
counter += 1
// Fix: delete these. They add visual clutter with no value.
```

---

## Data and Structure Smells

**Data clump** — group of fields that always travel together
```scala
// Smell: these three always appear together
def createInvoice(street: String, city: String, country: String, amount: BigDecimal): Invoice
// Fix: extract an object
case class Address(street: String, city: String, country: String)
def createInvoice(billingAddress: Address, amount: BigDecimal): Invoice
```

**Magic number / literal** — unexplained constant
```scala
// Smell:
if (items.length > 50) sendBulkEmail()
val price = amount * 1.21
// Fix:
val MAX_INDIVIDUAL_EMAIL_RECIPIENTS = 50
val EU_VAT_RATE = BigDecimal("0.21")
if (items.length > MAX_INDIVIDUAL_EMAIL_RECIPIENTS) sendBulkEmail()
val price = amount * (1 + EU_VAT_RATE)
```

**Primitive obsession** — using raw primitives instead of domain types
```scala
// Smell: Long for user ID and order ID — easy to mix up
def cancelOrder(userId: Long, orderId: Long): IO[Unit]
cancelOrder(orderId, userId)  // no compile error!

// Fix: value classes (zero runtime cost)
case class UserId(value: Long)  extends AnyVal
case class OrderId(value: Long) extends AnyVal
def cancelOrder(userId: UserId, orderId: OrderId): IO[Unit]
// cancelOrder(orderId, userId)  // compile error!
```

**Inconsistency** — same concept expressed differently in different places
```scala
// Smell: three representations of the same status concept
def findByStatus(status: String): List[Order]       // string
case class Order(status: Int, ...)                  // Int
val ACTIVE_STATUS = "active"                         // constant
// Fix: one canonical type
sealed trait OrderStatus
case object Active    extends OrderStatus
case object Cancelled extends OrderStatus
// Use OrderStatus everywhere
```

---

## Class Smells

**Low cohesion** — methods use different subsets of fields; signals multiple responsibilities
```scala
// Smell: half the methods use userRepo, half use paymentService
class UserService(
  userRepo: UserRepository,
  paymentService: PaymentService,
  emailService: EmailService,
  analyticsService: AnalyticsService
) {
  def findUser(id: Long)              = userRepo.find(id)          // uses userRepo only
  def processPayment(amount: BigDecimal) = paymentService.charge(amount)  // uses paymentService only
  def sendEmail(userId: Long, msg: String) = emailService.send(userId, msg)  // uses emailService only
  def trackEvent(event: String)       = analyticsService.track(event)  // uses analyticsService only
}
// Fix: each sub-service becomes its own class; cohesion is 100%
```

**SRP violation** — multiple actors (see SOLID → SRP in clean-architecture.md)

**Hybrid data/object** — exposes data AND provides operations (confusing to callers)
```scala
// Smell: Order is half data structure, half service object
case class Order(id: UUID, total: BigDecimal, status: String) {
  def validate(): Boolean = total > 0 && status.nonEmpty  // logic in data class
  def getStatus()         = status                         // redundant accessor
}

// Fix: pure data structure + pure functions
case class Order(id: UUID, total: BigDecimal, status: OrderStatus)
def validateOrder(order: Order): Either[ValidationError, Order] = ???
def isActive(order: Order): Boolean = order.status == Active
```

---

## General Smells

**Duplication (DRY violation)** — same logic appears in multiple places
```scala
// Smell: tax calculation appears in 3 places
val invoiceTotal = subtotal * 1.21
val orderTotal   = baseAmount * 1.21
val quoteTotal   = net * 1.21
// Fix: extract
val EU_VAT_RATE = BigDecimal("0.21")
def applyVAT(amount: BigDecimal) = amount * (1 + EU_VAT_RATE)
```

**Dead code** — code that is never executed
```scala
// Smell: after a guard/return, remaining code is unreachable
def process(order: Option[Order]): String = order match {
  case None        => "empty"
  case Some(o)     => o.id.toString
  case _ => "unreachable"  // dead code
}
// Fix: remove the unreachable case (sealed trait exhaustive check helps here)
```

**Vertical separation** — variable declared far from where it's used
```scala
// Smell:
val taxRate = 0.21                  // declared at top
// ... 30 lines of unrelated code ...
val total = price * (1 + taxRate)   // used at bottom
// Fix: declare taxRate immediately before total
val total = price * (1 + 0.21)  // or extract to a named constant nearby
```

**Artificial coupling** — code placed together for convenience, not for cohesion
```scala
// Smell: unrelated utilities in the same object
object Utils {
  def parseEmail(raw: String): Option[String]     // string utility
  def calculateVAT(amount: BigDecimal): BigDecimal // finance utility
  def formatDate(d: LocalDate): String             // date utility
  def sendEmail(to: String, body: String): Unit    // side-effecting I/O
}
// Fix: separate by domain
object EmailUtils { def parse(raw: String): Option[String] }
object TaxCalculator { def applyVAT(amount: BigDecimal): BigDecimal }
object DateFormatter { def format(d: LocalDate): String }
```

**Hidden temporal coupling** — two calls must happen in order but nothing enforces it
```scala
// Smell: caller must know to call init() before process()
class ReportGenerator {
  def init(): Unit = ???
  def process(): Report = ???  // crashes if init() wasn't called
}
// Fix: encode the sequence in the type
class ReportGenerator {
  def init(): InitializedGenerator = new InitializedGenerator()
}
class InitializedGenerator {
  def process(): Report = ???  // can only be called after init(), enforced by type
}
```

---

## Test Smells

**Fragile test** — breaks on unrelated changes (tests implementation, not behavior)
```scala
// Smell: tests internal implementation detail (specific method was called)
it("calls repository.save exactly once") {
  val mockRepo = mock[OrderRepository]
  service.processOrder(order)
  verify(mockRepo, times(1)).save(order)
}
// This breaks if you rename save() to persist() or refactor internals
// Fix: test the observable behavior
it("saves the order") {
  val inMemRepo = new InMemoryOrderRepository()
  service.processOrder(order)
  inMemRepo.find(order.id) shouldBe Some(order)
}
```

**Slow unit test** — depends on database, network, or file system in a unit test
```scala
// Smell:
it("validates user") {
  val user = userFromDatabase(42L)  // real DB call in unit test
  validateUser(user) shouldBe Right(user)
}
// Fix: validate is a pure function; no DB needed
it("validates user with valid data") {
  val user = User(42L, "Alice", "alice@example.com")
  validateUser(user) shouldBe Right(user)
}
```

**Non-deterministic test** — passes sometimes, fails sometimes
```scala
// Smell: depends on system time
it("marks order as expired") {
  val order = Order(id = 1L, createdAt = Instant.now().minusSeconds(100))
  isExpired(order) shouldBe true
}
// If Instant.now() is injected or passed as parameter:
it("marks order as expired") {
  val now   = Instant.parse("2024-01-15T10:00:00Z")
  val order = Order(id = 1L, createdAt = now.minusSeconds(100))
  isExpired(order, now) shouldBe true  // deterministic
}
```

**Testing private methods** — testing implementation, not contract
```scala
// Smell:
class OrderService { private def computeDiscount(t: UserType): Double = ... }
// Use reflection or make method package-private to test it
// Fix: test via the public method that uses it
it("processOrder applies VIP discount") {
  val result = service.processOrder(order, VIP)
  result.total shouldBe order.total * 0.9  // public behavior, not private method
}
```

**Multiple concepts in one test** (see testing.md for BUILD-OPERATE-CHECK)
```scala
// Smell: three things tested at once — when it fails, which one broke?
it("processes order") {
  val result = service.processOrder(order, VIP)
  result.status shouldBe Confirmed           // concept 1
  result.total shouldBe order.total * 0.9   // concept 2
  emailWasSent shouldBe true                 // concept 3
}
// Fix: one test per concept
it("confirms the order after processing") { ... }
it("applies VIP discount") { ... }
it("sends confirmation email") { ... }
```
