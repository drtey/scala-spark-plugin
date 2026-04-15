# Domain Modeling in Scala

## Case Classes: What Goes Inside

A case class is a pure data container. Put in it only the essential fields of the concept being modeled. Do not put mutable state, non-deterministic fields (timestamps generated at construction), or behavior.

```scala
// Right — models the concept cleanly
case class Order(id: OrderId, customerId: CustomerId, amount: Money, status: OrderStatus)

// Wrong — mixes business logic into the data container
case class Order(id: Long, amount: Double) {
  def isValid: Boolean = amount > 0 && id > 0  // belongs in a validator, not here
}

// Wrong — non-deterministic default breaks equality and test reproducibility
case class Order(id: OrderId, createdAt: Instant = Instant.now())
```

Rules:
- All fields `val` — case classes are immutable
- Avoid default values unless they represent a domain truth, not a convenience
- No side-effecting methods — case classes are data, not services

---

## Sealed Traits: Modeling Exhaustive Alternatives

Use a sealed trait when a type has a fixed, closed set of variants. The compiler enforces exhaustive pattern matching.

```scala
sealed trait OrderStatus
case object Pending   extends OrderStatus
case object Confirmed extends OrderStatus
case object Shipped   extends OrderStatus
case class  Cancelled(reason: String) extends OrderStatus

def describe(status: OrderStatus): String = status match {
  case Pending          => "awaiting confirmation"
  case Confirmed        => "confirmed"
  case Shipped          => "on its way"
  case Cancelled(why)   => s"cancelled: $why"
  // Missing a case → compiler warning: match may not be exhaustive
}
```

**sealed trait vs enum (Scala 3):**

```scala
// Use enum when all variants are pure labels — no extra fields
enum Direction:
  case North, South, East, West

// Use sealed trait when at least one variant carries data
sealed trait PaymentResult
case class Success(transactionId: TransactionId) extends PaymentResult
case class Failure(reason: String, retryable: Boolean) extends PaymentResult
```

---

## Make Illegal States Unrepresentable

Model your types so that invalid combinations cannot be constructed.

```scala
// Bad — allows User with both email and googleId, or neither
case class User(name: String, email: Option[String], googleId: Option[String])

// Right — each variant carries exactly the fields it needs
sealed trait UserIdentity
case class EmailUser(name: String, email: Email)        extends UserIdentity
case class GoogleUser(name: String, googleId: GoogleId) extends UserIdentity
case class GuestUser(sessionId: SessionId)              extends UserIdentity
```

Build types around what you know to be true, not around what might be present. An `Option[A]` in a case class is a flag that the type is incomplete — consider whether a sum type expresses the intent better.

---

## Avoid Primitive Obsession

Primitive types (`Long`, `String`, `Double`) carry no domain meaning. Two `Long` values can be silently swapped at the call site.

Use **opaque types** (Scala 3) to give distinct compile-time identities with zero runtime cost:

```scala
// Bad — compiler cannot distinguish userId from orderId
def assignOrder(userId: Long, orderId: Long): Unit = ???
assignOrder(orderId, userId)  // silently wrong — args swapped

// Right — distinct types, same runtime representation
object Domain {
  opaque type UserId  = Long
  opaque type OrderId = Long

  object UserId  { def apply(v: Long): UserId  = v }
  object OrderId { def apply(v: Long): OrderId = v }
}

import Domain._
def assignOrder(userId: UserId, orderId: OrderId): Unit = ???
// assignOrder(OrderId(1), UserId(2))  // compile error
```

Prefer opaque types over `extends AnyVal` in Scala 3: they are strictly enforced at the file boundary and do not have the boxing edge cases of value classes.

---

## Smart Constructors

Prevent invalid values from ever being created by making the constructor private and validating in a factory method.

```scala
// Bad — Email accepts any string
case class Email(value: String)
val invalid = Email("not-an-email")  // compiles

// Right — construction goes through validated factory
final case class Email private (value: String)

object Email {
  def parse(raw: String): Either[String, Email] =
    if (raw.contains("@") && raw.length <= 254) Right(Email(raw))
    else Left(s"Invalid email: $raw")
}

// Consumers must handle the Either — invalid emails cannot slip through
Email.parse(input).map(sendWelcomeEmail)
```

Use `Either[String, A]` for single-field validation. Use `ValidatedNel[String, A]` (Cats) to accumulate errors across multiple fields simultaneously:

```scala
import cats.data.ValidatedNel
import cats.syntax.validated._
import cats.syntax.apply._

def validateName(s: String): ValidatedNel[String, Name] =
  if (s.nonEmpty) Name(s).validNel else "Name cannot be empty".invalidNel

def validateEmail(s: String): ValidatedNel[String, Email] =
  Email.parse(s).toValidatedNel

def validateUser(name: String, email: String): ValidatedNel[String, User] =
  (validateName(name), validateEmail(email)).mapN(User.apply)
// Returns both errors if both fail — not just the first
```

---

## Immutable Domain Updates

Use `copy` for single-field updates. For deeply nested structures, extract named helpers or use Monocle lenses.

```scala
// Simple — copy is correct and readable
val shipped = order.copy(status = Shipped)

// Nested update without Monocle
case class Address(street: String, city: String)
case class Customer(name: String, address: Address)

val moved = customer.copy(address = customer.address.copy(city = "Madrid"))

// Deeply nested — Monocle lens (Scala 3)
import monocle.syntax.all._
val moved = customer.focus(_.address.city).replace("Madrid")
```

If you find yourself writing `a.copy(b = a.b.copy(c = a.b.c.copy(...)))` more than once, either introduce lenses or flatten the data structure.
