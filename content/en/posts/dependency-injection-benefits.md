---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Dependency Injection + Dependency Inversion: More Robust and Testable Code"
date: 2025-08-22T08:00:00+01:00
description: "How applying the Dependency Inversion Principle from SOLID together with a Dependency Injection framework leads to decoupled, robust, and highly testable code."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/di-fix.png
draft: false
tags:
- architecture
- solid
- dependency-injection
- testing
---

### Dependency Injection + Dependency Inversion: More Robust and Testable Code

Modern applications evolve quickly: features are added, platforms multiply, and teams scale. In this environment, tightly coupled code becomes a bottleneck for change and a source of fragile tests. Two key ideas help us fight this complexity:

- Dependency Inversion Principle (DIP) from SOLID: high-level modules should not depend on low-level modules. Both should depend on abstractions.
- Dependency Injection (DI): a technique and set of tools to provide those dependencies from the outside instead of constructing them internally.

In this post, we’ll see why DIP matters, how DI enforces it, and how using a DI framework makes your code more robust and testable—with Kotlin-flavored examples that apply equally to other languages.

---

#### Dependency Inversion in a Nutshell (the “D” in SOLID)

DIP encourages designing around abstractions (interfaces) rather than concrete implementations. The goal is to decouple business logic (high-level) from infrastructure details (low-level):

- High-level modules depend on interfaces they define or control
- Low-level modules implement those interfaces
- Composition happens at the boundaries (app startup, DI container, factories)

This reduces ripple effects of change and makes substitution and testing straightforward.

```kotlin
// High-level policy
interface PaymentProcessor {
    suspend fun charge(amount: Money): Result<Unit>
}

// Low-level implementation
class StripePaymentProcessor(private val api: StripeApi) : PaymentProcessor {
    override suspend fun charge(amount: Money): Result<Unit> = api.charge(amount)
}
```

---

#### What Dependency Injection Adds

DI is the practice of providing concrete implementations to components from the outside. Instead of constructing dependencies with `new` (or constructors) inside a class, we accept them via constructor parameters (or setters), and a composition root (often a DI framework) wires the graph. Benefits:

- Decoupling: classes depend on interfaces, not on how to create them
- Testability: dependencies can be replaced with fakes/mocks easily
- Single Responsibility: classes focus on behavior, not object creation
- Replaceability: swap implementations without touching consumers

```kotlin
class CheckoutService(private val paymentProcessor: PaymentProcessor) {
    suspend fun checkout(cart: Cart): Result<Unit> {
        // business logic only
        return paymentProcessor.charge(cart.total())
    }
}
```

No `CheckoutService` knowledge about Stripe, PayPal, or HTTP clients—only the abstraction.

---

#### Before vs After: A Concrete Example

Without DIP/DI (tightly coupled):

```kotlin
class CheckoutService {
    private val api = StripeApi(httpClient = HttpClient())
    private val processor = StripePaymentProcessor(api)

    suspend fun checkout(cart: Cart): Result<Unit> {
        // Hard to test: real network and Stripe specifics leak in
        return processor.charge(cart.total())
    }
}
```

Problems:
- Hard to unit test (requires network or heavy mocking tools)
- Changing provider (Stripe -> PayPal) touches this class
- Violates SRP by mixing construction and business rules

With DIP + DI:

```kotlin
class CheckoutService(private val paymentProcessor: PaymentProcessor) {
    suspend fun checkout(cart: Cart): Result<Unit> = paymentProcessor.charge(cart.total())
}
```

Composition (using any DI style):

```kotlin
val api = StripeApi(HttpClient())
val processor: PaymentProcessor = StripePaymentProcessor(api)
val service = CheckoutService(processor)
```

Test becomes trivial:

```kotlin
class FakePaymentProcessor : PaymentProcessor {
    var lastAmount: Money? = null
    override suspend fun charge(amount: Money): Result<Unit> {
        lastAmount = amount
        return Result.success(Unit)
    }
}

@Test
fun `checkout charges cart total`() = runTest {
    val fake = FakePaymentProcessor()
    val service = CheckoutService(fake)

    val result = service.checkout(cart = Cart(listOf(Line("abc", 100))))

    assertTrue(result.isSuccess)
    assertEquals(Money(100), fake.lastAmount)
}
```

---

#### Using a DI Framework: Why It Helps

As your application grows, manual wiring becomes error-prone. DI frameworks act as a composition engine:

- Centralize object graphs and lifecycles (singleton, scoped, transient)
- Enforce explicit dependencies (constructor injection)
- Provide compile-time or runtime validation

Common options by ecosystem:

- Kotlin/Android: Hilt/Dagger (compile-time, annotations), Koin (DSL), Kodein
- JVM/backend: Spring Framework/Spring Boot (annotations), Guice
- Multiplatform: Koin works for KMP; some projects use manual composition or Service Locator for shared code

Example with Koin (simple DSL):

```kotlin
val appModule = module {
    single { HttpClient() }
    single { StripeApi(get()) }
    single<PaymentProcessor> { StripePaymentProcessor(get()) }
    factory { CheckoutService(get()) }
}

// Start Koin
startKoin { modules(appModule) }

// Resolve where needed (constructor injection recommended in real apps)
val service: CheckoutService = get()
```

Example with Hilt (Android):

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object PaymentsModule {
    @Provides fun provideHttpClient(): HttpClient = HttpClient()
    @Provides fun provideStripeApi(client: HttpClient) = StripeApi(client)
    @Provides fun provideProcessor(api: StripeApi): PaymentProcessor = StripePaymentProcessor(api)
}

@HiltViewModel
class CheckoutViewModel @Inject constructor(
    private val service: CheckoutService
) : ViewModel() {
    // ...
}
```

---

#### Robustness and Testability in Practice

- Smaller blast radius: infrastructure changes don’t cascade through business logic
- Deterministic tests: inject fakes; no hidden singletons or global state
- Clear boundaries: interfaces model the seams of your system
- Easier maintenance: constructor parameters make dependencies visible and reviewable

Checklists:

- Do my high-level modules depend only on abstractions they own?
- Can I swap implementations without editing consumers?
- Are constructors the only way dependencies enter my types?
- Can I unit test without network, disk, or threads?

---

#### Common Pitfalls and How to Avoid Them

- Service Locator antipattern: avoid pulling dependencies from a global registry inside classes; prefer constructor injection
- God modules: split DI modules by feature/boundary to avoid giant graphs
- Over-abstraction: don’t create interfaces without alternate implementations or test value
- Hidden state: avoid static singletons; prefer DI-managed lifecycles

---

#### Conclusion

Applying the Dependency Inversion Principle and adopting Dependency Injection yields code that is easier to maintain, extend, and test. Start small: define interfaces for your boundaries, inject dependencies via constructors, and gradually introduce a DI framework as your graph grows. The payoff is significant—cleaner architecture, faster tests, and safer change.
