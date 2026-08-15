# Seams: Where to Insert Tests Into Legacy Code

> Based on Michael Feathers, *Working Effectively with Legacy Code* (2004), Chapter 4 — with Kotlin/JVM examples

## TL;DR

A **seam** is a place in your code where you can alter behavior **without editing the code at that spot**. Legacy code usually resists testing because it's tangled up with production dependencies (databases, networks, the system clock, global state), and rewriting all of that by hand — without tests to protect you — is risky. Seams solve this dilemma: instead of *changing code to make it testable*, you find an existing point where you can intercept execution.

Every seam has two parts:

- **The seam itself**: the point where execution can be intercepted
- **The enabling point**: the place where you actually choose which behavior to use (in Kotlin, this is almost always the constructor call site)

---

## 1. The Three Kinds of Seams

Feathers classifies seams into three types. In practice, the first two account for the vast majority of real-world use.

| Type | Where it intervenes | Typical JVM/Kotlin form |
|---|---|---|
| Object seam | At runtime, when the object graph is assembled | Interface extraction, constructor injection, higher-order function parameters |
| Preprocessing seam | At compile time / bytecode generation | Bytecode manipulation (ByteBuddy, ASM), AspectJ weaving |
| Link seam | At classloading / link time | DI container profiles, ServiceLoader, classpath ordering |

### 1.1 Object seam — the safest and most common

Kotlin/JVM is object-oriented, so the overwhelming majority of seams you'll use are object seams. A useful mental model: **wherever polymorphism exists, a seam exists**.

```kotlin
// Before — no seam. To test OrderProcessor you need a real Postgres instance.
class OrderProcessor {
    fun process(order: Order): Receipt {
        val db = PostgresConnection.getInstance()
        db.save(order)
        return Receipt(order.id, LocalDateTime.now())
    }
}

// After — extract an interface to create an object seam
interface OrderStore {
    fun save(order: Order)
}

class OrderProcessor(
    private val store: OrderStore,
) {
    fun process(order: Order): Receipt {
        store.save(order)
        return Receipt(order.id, LocalDateTime.now())
    }
}
```

Here `store` is the seam, and the call site that constructs `OrderProcessor(...)` is the enabling point. Production code assembles `OrderProcessor(PostgresOrderStore())`, tests assemble `OrderProcessor(FakeOrderStore())` — and the internal logic of `OrderProcessor.process()` was never touched.

### 1.2 Preprocessing seam — a last resort on Kotlin/JVM

The classic example is C/C++ `#define` macros. Kotlin has no compile-time macro system, so this category barely exists at the language level. What you find instead are bytecode-manipulation tools achieving a similar effect.

```kotlin
// Using ByteBuddy to intercept a static method call
// (only worth considering when a static factory/singleton is too deeply
// embedded to fix with an object seam)
val agent = ByteBuddyAgent.install()
new ByteBuddy()
    .redefine(PostgresConnection::class.java)
    .method(named("getInstance"))
    .intercept(MethodDelegation.to(FakeConnectionInterceptor::class.java))
    .make()
    .load(PostgresConnection::class.java.classLoader, ClassReloadingStrategy.fromInstalledAgent())
```

This approach slows tests down, ties them tightly to the execution environment, and makes it hard for teammates to tell what's actually happening just by reading the code. **Reaching for a preprocessing seam when an object seam would work is an anti-pattern.** Reserve it for cases where you genuinely cannot touch the source — e.g., a third-party binary dependency.

### 1.3 Link seam — practical in DI-container environments

A link seam swaps which implementation gets used at classloading/link time. On Kotlin/JVM it typically shows up as:

```kotlin
// Spring: swap implementations via profiles (a practical form of link seam)
@Service
@Profile("!test")
class StripePaymentGateway : PaymentGateway { /* ... */ }

@Service
@Profile("test")
class FakePaymentGateway : PaymentGateway { /* ... */ }
```

```kotlin
// ServiceLoader-based link seam — works even without a DI framework
// Swap the contents of META-INF/services/com.example.PaymentGateway
// on the test classpath only
val gateway: PaymentGateway = ServiceLoader.load(PaymentGateway::class.java).first()
```

The defining trait of a link seam is that the swap point isn't visible "in the code" — the enabling point has moved out to classpath configuration or container config files, which makes it harder for someone reading the test to trace which implementation is actually in play, compared to an object seam. Prefer expressing things as object seams where you can, and reach for link seams only when you need large-scale switching across multiple modules.

---

## 2. Five Patterns for Creating an Object Seam

### 2.1 Extract Interface

The most standard technique. Change a dependency on a concrete class into a dependency on an interface.

```kotlin
// Before
class ReportGenerator {
    private val mailer = SmtpMailer("smtp.internal", 25)
    fun sendReport(report: Report) = mailer.send(report.toEmail())
}

// After
interface Mailer { fun send(email: Email) }
class SmtpMailer(host: String, port: Int) : Mailer { /* ... */ }

class ReportGenerator(private val mailer: Mailer) {
    fun sendReport(report: Report) = mailer.send(report.toEmail())
}
```

### 2.2 Parameterize Constructor

When a legacy class directly constructs its own dependency internally, pull that construction logic out into a parameter.

```kotlin
// Before — constructed internally, cannot be swapped
class InvoiceService {
    fun issue(order: Order): Invoice {
        val taxCalculator = KoreaTaxCalculator()  // hard dependency
        return Invoice(order, taxCalculator.calculate(order))
    }
}

// After
class InvoiceService(
    private val taxCalculator: TaxCalculator = KoreaTaxCalculator()  // default preserves backward compatibility
) {
    fun issue(order: Order): Invoice = Invoice(order, taxCalculator.calculate(order))
}
```

Using a default parameter means you don't have to touch any existing call sites (enabling points) at all. This is one of Kotlin's advantages over Java, where you'd typically need a separate overloaded constructor to achieve the same thing.

### 2.3 Use a Function as the Seam (Higher-order function seam)

Especially useful in Kotlin. When extracting an entire class into an interface feels like overkill, you can pull out just the one problematic behavior as a function type. The `LocalDateTime.now()` case from Example 1 is a typical instance of this.

```kotlin
class OrderProcessor(
    private val store: OrderStore,
    private val clock: () -> LocalDateTime = { LocalDateTime.now() },
    private val idGenerator: () -> String = { UUID.randomUUID().toString() },
) {
    fun process(order: Order): Receipt {
        store.save(order)
        return Receipt(idGenerator(), order.id, clock())
    }
}
```

In tests, injecting `clock = { fixedInstant }` gives you a deterministic value, eliminating flakiness caused by "what time is it right now."

### 2.4 Wrap — when you can't change the existing class

When a legacy class is `final`, or comes from a third-party library you can't subclass or modify, build a thin wrapper and put the seam behind that.

```kotlin
// A third-party SDK class is final and can't be mocked directly
interface HttpClientPort {
    fun get(url: String): String
}

class OkHttpClientAdapter(private val client: OkHttpClient) : HttpClientPort {
    override fun get(url: String): String =
        client.newCall(Request.Builder().url(url).build()).execute().body!!.string()
}
```

Now `HttpClientPort` is the seam, `OkHttpClientAdapter` is the production enabling point, and tests use a `FakeHttpClientPort`.

### 2.5 Turn Static Access Into Instance Access (avoid "Introduce Static Setter")

Feathers' original book also describes using a static setter to swap out global state at test time, but this tends to break test isolation and isn't recommended in real-world Kotlin/JVM codebases. Instead, hide the singleton/`object` behind an interface and switch to constructor injection.

```kotlin
// Avoid — creating a seam via global mutable state
object FeatureFlags {
    var overrideForTest: Boolean? = null  // risk of cross-test pollution
}

// Prefer — hide the object behind an interface and inject it
interface FeatureFlagProvider { fun isEnabled(flag: String): Boolean }
```

---

## 3. A Process for Finding Seams in Legacy Code

1. **List what the method you want to test depends on.** Database, filesystem, network, the clock, randomness, global mutable state, static factory calls.
2. **Check whether each dependency is already polymorphic.** If an interface or superclass already exists, that's already a seam — you just need to build a fake against it, no extra work required.
3. **If it's not polymorphic, extract the smallest possible unit.** Before pulling an entire class out into an interface, first check whether you can extract just the one problematic method as a function-type parameter (see §2.3).
4. **Verify the enabling point.** A seam is useless if there's still no way to inject a fake at the actual call site (typically production assembly code, `main`, or DI configuration). If constructor injection isn't possible due to the existing structure (static factories, singletons), consider a link seam (§1.3) or a wrap (§2.4).
5. **Start with the lowest-risk changes.** Interface extraction can often be done safely with automated IDE refactoring. Start with these "mechanically safe" changes (what Feathers calls a "safe change"), and defer any change to actual logic until the seams you need are fully in place.

---

## 4. Common Pitfalls

- **Don't create too many seams too early.** Turning every dependency into an interface leads to over-engineering. Only create seams where you actually need them — i.e., where you intend to write a characterization test.
- **Don't confuse a seam with an abstraction.** A seam's purpose is testability, not reusability. Interfaces created "just in case" only add maintenance cost.
- **Don't create seams via global mutable state.** As noted in §2.5, if test isolation breaks down, the seam itself becomes a new source of bugs.
- **Don't overuse link seams.** As more of the codebase reaches a state where you can't tell from reading the code alone which implementation is in play, you end up with code that's harder to read than the legacy code you started with.

---

## Reference

- Michael Feathers, *Working Effectively with Legacy Code*, Chapter 4: "The Seam Model" (Prentice Hall, 2004)
