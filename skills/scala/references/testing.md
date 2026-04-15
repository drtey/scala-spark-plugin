# TDD Reference for Scala

## F.I.R.S.T. Principles — Properties of a Good Test

```
F — Fast
    Tests must run in milliseconds. If they're slow, developers stop running them.
    Rule: unit tests hit no database, no network, no filesystem. IO belongs in integration tests.

I — Independent
    Each test sets up its own fixtures and tears down its own state.
    No test should depend on a previous test having run first.
    Rule: never share mutable state between tests. Use BeforeAndAfterEach, not BeforeAndAfterAll,
    for state that must be reset.

R — Repeatable
    The test produces the same result every time, in any environment, at any time.
    Violations: System.currentTimeMillis(), new Random(), external HTTP calls,
    hard-coded file paths, race conditions in parallel tests.
    Rule: inject a Clock, a Seed, a stub HTTP client. Control all sources of non-determinism.

S — Self-Validating
    The test asserts its own pass/fail. No manual log inspection, no "check if the file appeared".
    shouldBe, assertEquals, assert — the test runner knows without human interpretation.
    A test without assertions is a lie: it always passes.

T — Timely
    Write tests at the same time as or just before the production code (TDD).
    Tests written after the code are optimistic — you already know it works, so you
    unconsciously test only the happy path.
```

---

## BUILD-OPERATE-CHECK Pattern

```scala
it("applies VIP discount correctly") {
  // BUILD — create everything the test needs, and nothing else
  val order = Order(
    id     = UUID.fromString("00000000-0000-0000-0000-000000000001"),
    items  = List(OrderItem("SKU-1", quantity = 2, unitPrice = BigDecimal(50))),
    total  = BigDecimal(100)
  )

  // OPERATE — call exactly one function under test
  val result = applyDiscount(VIP, order)

  // CHECK — assert the single observable outcome
  result.total shouldBe BigDecimal(90)
}
```

**What BUILD-OPERATE-CHECK prevents:**

```scala
// Anti-pattern: setup, operation, and assertions interleaved
it("processes order") {
  val repo = new InMemoryOrderRepository()   // setup
  val svc  = new OrderService(repo)
  val order = makeOrder()                    // setup
  val result = svc.processOrder(order)       // operate
  repo.findAll().size shouldBe 1             // check 1
  result.status shouldBe OrderStatus.Confirmed  // check 2 — which one failed?
  result.total shouldBe BigDecimal(90)       // check 3 — can't tell from the name
}

// Fixed: one BUILD-OPERATE-CHECK per concept
it("persists the order after processing") {
  val repo = new InMemoryOrderRepository()
  val svc  = OrderService(repo)
  svc.processOrder(makeOrder())
  repo.findAll() should have size 1          // only this assertion
}

it("confirms the order status") {
  val result = OrderService(new InMemoryOrderRepository()).processOrder(makeOrder())
  result.status shouldBe OrderStatus.Confirmed
}

it("applies VIP discount to the total") {
  val result = OrderService(new InMemoryOrderRepository()).processOrder(makeOrder())
  result.total shouldBe BigDecimal(90)
}
```

---

## One Assert Per Test

```scala
// Bad: three concepts in one test — which one failed?
it("validates user") {
  validateUser(User("", "alice@example.com", 25)) shouldBe Left(NameEmpty)
  validateUser(User("Alice", "not-an-email", 25)) shouldBe Left(EmailInvalid)
  validateUser(User("Alice", "alice@example.com", 15)) shouldBe Left(AgeTooYoung)
}

// Good: one concept per test — failure name = diagnosis
it("rejects empty name") {
  validateUser(User("", "alice@example.com", 25)) shouldBe Left(NameEmpty)
}
it("rejects invalid email format") {
  validateUser(User("Alice", "not-an-email", 25)) shouldBe Left(EmailInvalid)
}
it("rejects age below 18") {
  validateUser(User("Alice", "alice@example.com", 15)) shouldBe Left(AgeTooYoung)
}
it("accepts a valid user") {
  validateUser(User("Alice", "alice@example.com", 25)) shouldBe Right(User("Alice", "alice@example.com", 25))
}
```

**One concept ≠ one line.** Sometimes verifying one concept requires multiple assertions:

```scala
// Acceptable: multiple assertions, all verifying the same concept (order confirmation)
it("confirms the order") {
  val result = service.confirmOrder(pendingOrder)
  result.status shouldBe Confirmed
  result.confirmedAt should not be empty   // both assertions describe "confirmation"
}
```

---

## The TDD Cycle

Red (write a failing test) -> Green (write minimum code to pass) -> Refactor (clean up without breaking).

```scala
// Step 1: RED — write the test first
class DiscountSpec extends AnyFunSpec with Matchers {
  describe("applyDiscount") {
    it("reduces price by 10% for VIP users") {
      applyDiscount(VipUser, BigDecimal(100)) shouldBe BigDecimal(90)
    }
    it("does not change price for regular users") {
      applyDiscount(RegularUser, BigDecimal(100)) shouldBe BigDecimal(100)
    }
  }
}

// Step 2: GREEN — minimum code to make tests pass
def applyDiscount(user: User, price: BigDecimal) = user match {
  case VipUser     => price * BigDecimal("0.9")
  case RegularUser => price
}

// Step 3: REFACTOR — extract, rename, simplify
// TDD forces pure functions: if it's hard to test, it has hidden dependencies
```

---

## ScalaTest — Core DSL

### Test Styles

```scala
// AnyFunSpec — BDD-style, describe/it, best for behavior documentation
class OrderServiceSpec extends AnyFunSpec with Matchers {
  describe("OrderService") {
    describe("calculateTotal") {
      it("sums line items") { ??? }
      it("applies tax") { ??? }
    }
  }
}

// AnyFlatSpec — flat unit tests, concise
class OrderSpec extends AnyFlatSpec with Matchers {
  "Order.total" should "sum all line items" in { ??? }
  it should "return zero for empty order" in { ??? }
}

// AnyWordSpec — hierarchical, verbose
class UserSpec extends AnyWordSpec with Matchers {
  "User" when {
    "validating email" should {
      "reject emails without @" in { ??? }
    }
  }
}
```

### Matchers DSL

```scala
// Equality
result shouldBe 42
result shouldEqual "expected"
result should not be null

// Numeric
result should be > 0
result should be >= 100
result shouldBe 3.14 +- 0.001  // tolerance

// Collections
list should have length 3
list should contain("item")
list should contain allOf ("a", "b", "c")
list should contain only ("a", "b")
list shouldBe sorted
map should contain key "name"
map should contain value 42
seq should be(empty)
seq should not be empty

// Option / Either
option shouldBe defined
option shouldBe empty
option.value shouldBe "expected"  // asserts Some and extracts

either shouldBe Symbol("right")
either.value shouldBe 42          // asserts Right and extracts

// Exceptions
a[IllegalArgumentException] should be thrownBy { riskyFunction() }
the[RuntimeException] thrownBy { riskyFunction() } should have message "oops"

// Custom assertions
result should matchPattern { case User(_, "alice@example.com", _) => }
```

### Setup and Teardown

```scala
class DbSpec extends AnyFunSpec with Matchers with BeforeAndAfterEach {
  var connection: Connection = _

  override def beforeEach(): Unit = {
    connection = openTestConnection()
  }

  override def afterEach(): Unit = {
    connection.close()
  }

  it("inserts a record") {
    val count = insert(connection, testUser)
    count shouldBe 1
  }
}
```

---

## ScalaCheck — Property-Based Testing

```scala
import org.scalacheck.{Gen, Prop, Properties}
import org.scalacheck.Prop.{forAll, forAllNoShrink}

// ScalaCheck integrated with ScalaTest
import org.scalatestplus.scalacheck.ScalaCheckPropertyChecks

class DiscountProperties extends AnyFunSpec with ScalaCheckPropertyChecks with Matchers {
  describe("applyDiscount") {
    it("never increases the price") {
      forAll(Gen.posNum[Double]) { amount =>
        val price = BigDecimal(amount)
        forAll(Gen.oneOf(VipUser, RegularUser)) { user =>
          applyDiscount(user, price) should be <= price
        }
      }
    }

    it("VIP discount is exactly 10%") {
      forAll(Gen.posNum[Double]) { amount =>
        val price = BigDecimal(amount)
        applyDiscount(VipUser, price) shouldBe price * BigDecimal("0.9")
      }
    }
  }
}
```

### Generators

```scala
import org.scalacheck.Gen

// Primitives
val posInt    = Gen.posNum[Int]
val negInt    = Gen.negNum[Int]
val smallInt  = Gen.choose(1, 100)
val digit     = Gen.numChar

// Strings
val nonEmpty  = Gen.nonEmptyStr
val alphaStr  = Gen.alphaStr
val email     = for {
  local  <- Gen.alphaStr.suchThat(_.nonEmpty)
  domain <- Gen.alphaStr.suchThat(_.nonEmpty)
} yield s"$local@$domain.com"

// Collections
val listOf5    = Gen.listOfN(5, Gen.posNum[Int])
val nonEmpty2  = Gen.nonEmptyListOf(Gen.alphaChar)

// From enumeration
val userType   = Gen.oneOf(VipUser, RegularUser, GuestUser)
val status     = Gen.oneOf("active", "inactive", "pending")

// Case class generator
val genOrder = for {
  id     <- Gen.posNum[Long]
  amount <- Gen.choose(1.0, 10000.0)
  status <- Gen.oneOf("active", "cancelled")
} yield Order(id, BigDecimal(amount), status)

// Recursive / nested
val genNestedList = Gen.listOf(Gen.listOf(Gen.posNum[Int]))
```

### Properties

```scala
// Algebraic properties are ideal for PBT
// Commutativity
forAll { (a: Int, b: Int) => add(a, b) == add(b, a) }

// Associativity
forAll { (a: Int, b: Int, c: Int) => add(add(a, b), c) == add(a, add(b, c)) }

// Identity
forAll { (a: Int) => add(a, 0) == a }

// Round-trip: encode then decode returns original
forAll(genOrder) { order =>
  decode(encode(order)) == Right(order)
}

// Idempotency
forAll(genOrder) { order =>
  deduplicate(deduplicate(order)) == deduplicate(order)
}

// Exhaustive sealed trait — compiler proves you covered all cases
// No need to generate: sealed trait + pattern match = compile-time exhaustion check
```

---

## Testing Sealed Traits — Exhaustive Coverage

```scala
sealed trait PaymentMethod
case object CreditCard extends PaymentMethod
case object BankTransfer extends PaymentMethod
case object Crypto extends PaymentMethod

// If you add WireTransfer to PaymentMethod and forget to handle it,
// the compiler warns on the match — no test case needed for the case itself
def processFee(method: PaymentMethod): BigDecimal = method match {
  case CreditCard    => BigDecimal("0.029")
  case BankTransfer  => BigDecimal("0.001")
  case Crypto        => BigDecimal("0.005")
  // Adding WireTransfer without a case here → compile warning
}

// Test each case explicitly — the sealed trait guarantees exhaustion
class FeeSpec extends AnyFunSpec with Matchers {
  describe("processFee") {
    it("charges 2.9% for credit card") {
      processFee(CreditCard) shouldBe BigDecimal("0.029")
    }
    it("charges 0.1% for bank transfer") {
      processFee(BankTransfer) shouldBe BigDecimal("0.001")
    }
    it("charges 0.5% for crypto") {
      processFee(Crypto) shouldBe BigDecimal("0.005")
    }
  }
}
```

---

## Testing Pure vs Impure Code

Strategy: test the pure core exhaustively; test the impure edges minimally.

```scala
// Layer 1: PURE CORE — test without any mocks or infrastructure
object OrderLogic {
  def applyTax(order: Order, rate: BigDecimal) =
    order.copy(total = order.total * (1 + rate))

  def validate(order: Order): Either[String, Order] =
    if (order.total <= 0) Left("Total must be positive")
    else Right(order)
}

class OrderLogicSpec extends AnyFunSpec with Matchers {
  val validOrder = Order(id = 1L, total = BigDecimal(100))

  describe("applyTax") {
    it("adds the tax rate to the total") {
      val result = OrderLogic.applyTax(validOrder, BigDecimal("0.1"))
      result.total shouldBe BigDecimal(110)
    }
  }

  describe("validate") {
    it("rejects orders with zero total") {
      val result = OrderLogic.validate(validOrder.copy(total = BigDecimal(0)))
      result shouldBe Left("Total must be positive")
    }
    it("accepts orders with positive total") {
      OrderLogic.validate(validOrder) shouldBe Right(validOrder)
    }
  }
}

// Layer 2: IMPURE EDGE — test the integration point separately
// Use a test double only when you must cross a boundary
trait OrderRepository {
  def save(order: Order): Either[String, Long]
}

class InMemoryOrderRepository extends OrderRepository {
  private var store = Map.empty[Long, Order]
  def save(order: Order) = {
    store = store + (order.id -> order)
    Right(order.id)
  }
  def find(id: Long) = store.get(id)
}

class OrderServiceSpec extends AnyFunSpec with Matchers {
  val repo    = new InMemoryOrderRepository
  val service = new OrderService(repo, OrderLogic)

  it("saves a valid order and returns its id") {
    val order  = Order(id = 1L, total = BigDecimal(50))
    val result = service.create(order)
    result shouldBe Right(1L)
    repo.find(1L) shouldBe Some(order)
  }
}
```

---

## Mocking with Mockito Scala

```scala
import org.mockito.MockitoSugar
import org.mockito.ArgumentMatchers._

class UserServiceSpec extends AnyFunSpec with Matchers with MockitoSugar {
  val mockRepo  = mock[UserRepository]
  val service   = new UserService(mockRepo)

  describe("findByEmail") {
    it("returns user when found") {
      val user = User(1L, "Alice", "a@b.com")
      when(mockRepo.findByEmail("a@b.com")).thenReturn(Some(user))

      service.findByEmail("a@b.com") shouldBe Some(user)
      verify(mockRepo).findByEmail("a@b.com")
    }

    it("returns None when not found") {
      when(mockRepo.findByEmail(any[String])).thenReturn(None)
      service.findByEmail("nobody@x.com") shouldBe None
    }
  }
}
```

**Prefer real implementations** (like `InMemoryOrderRepository`) over mocks — they test the real contract, not just your assumptions about it.

---

## TestContainers — Integration Tests with Real Infrastructure

```scala
import com.dimafeng.testcontainers.{PostgreSQLContainer, ForAllTestContainer}
import org.testcontainers.utility.DockerImageName

class UserRepoIntegrationSpec extends AnyFunSpec
    with Matchers
    with ForAllTestContainer {

  override val container = PostgreSQLContainer(DockerImageName.parse("postgres:15"))

  lazy val repo = new PostgresUserRepository(container.jdbcUrl, container.username, container.password)

  it("persists and retrieves a user") {
    val user   = User(0L, "Alice", "alice@example.com")
    val saved  = repo.save(user)
    val found  = repo.find(saved.id)
    found shouldBe Some(saved)
  }
}
```

---

## Cats Effect — Testing IO

```scala
import cats.effect.{IO, Resource}
import cats.effect.testing.scalatest.AsyncIOSpec

class EmailServiceSpec extends AsyncIOSpec with Matchers {
  // Returns IO — framework runs it for you
  it("sends email and returns message id") {
    val service = new EmailService(testSmtpConfig)
    for {
      id     <- service.send(Email("to@x.com", "subject", "body"))
      result <- service.status(id)
    } yield result shouldBe "delivered"
  }
}

// Testing Resource acquisition and release
class ConnectionSpec extends AsyncIOSpec with Matchers {
  it("releases connection after use") {
    var released = false
    val resource = Resource.make(IO("connection"))(_ => IO { released = true })
    for {
      _ <- resource.use(conn => IO(conn.length))
    } yield released shouldBe true
  }
}
```

---

## MUnit — Lightweight Alternative

```scala
import munit.FunSuite

class CalculatorSuite extends FunSuite {
  test("add returns correct sum") {
    assertEquals(add(2, 3), 5)
  }

  test("divide throws on zero") {
    intercept[ArithmeticException] {
      divide(10, 0)
    }
  }

  // Clues — better failure messages
  test("all items are positive") {
    val items = process(input)
    items.foreach { item =>
      assert(item > 0, clue(item))
    }
  }
}
```

---

## Test Organization Conventions

```
src/
  main/scala/com/example/
    domain/
      Order.scala          -- pure domain model + logic
      OrderService.scala   -- orchestrates pure logic + impure edges
    infrastructure/
      PostgresOrderRepo.scala  -- impure: real DB
  test/scala/com/example/
    domain/
      OrderSpec.scala          -- pure tests, no infrastructure
      OrderProperties.scala    -- ScalaCheck property tests
    infrastructure/
      PostgresOrderRepoSpec.scala  -- integration tests with TestContainers
    service/
      OrderServiceSpec.scala   -- service tests with in-memory repos
```

Rules:
1. Pure tests run without any setup — fast, always run in CI
2. Integration tests require Docker — gate behind a tag or separate task
3. Never test implementation details — test the public contract
4. One assertion per test — when a test fails, the name tells you exactly what broke
