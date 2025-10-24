---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Zero-Cost Abstractions in Kotlin: Inline Functions and Value Classes"
date: 2025-10-24T08:00:00+01:00
description: "A practical guide to Kotlin inline functions and value classes: how they work, why they matter, and when to use them for safer, faster code."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/inline.png
draft: false
tags: 
- kotlin
- inline
- value-classes
- performance
- generics
---

### Zero-Cost Abstractions in Kotlin: Inline Functions and Value Classes

Kotlin gives you two powerful tools to write safer and faster code with zero or near-zero runtime overhead: inline functions and value classes. Used correctly, they help you avoid allocations, improve type safety, and keep APIs expressive.

This post explains what they are, how they work under the hood, practical use cases, trade-offs, and when not to use them.

---

#### TL;DR

- Inline functions remove call-site overhead for small higher-order utilities and enable reified type parameters.
- Value classes wrap a single value with strong type safety and (often) zero allocation at runtime.
- Use inline for small, frequently called higher-order functions and utilities where reified types help.
- Use value classes for domain-specific types (UserId, Email, Money) to prevent mix-ups while keeping performance close to primitives.
- Watch out for value class boxing with generics and nullability, and for overusing inline on large functions.

---

#### Inline Functions: What and Why

An inline function asks the compiler to copy its body directly at each call site. This is particularly useful for higher-order functions (functions that take lambdas) because it can remove the allocation of function objects and the virtual call overhead.

Basic example:

```kotlin
inline fun <T> measure(label: String, block: () -> T): T {
    val start = systemTimeMillis()
    try {
        return block()
    } finally {
        val elapsed = systemTimeMillis() - start
        log("$label took ${elapsed}ms")
    }
}

fun demo() {
    val result = measure("expensive") {
        computeHeavy()
    }
}
```

Key modifiers:

- noinline: prevents a specific lambda parameter from being inlined.
- crossinline: forbids non-local returns from a lambda (useful when you re-invoke the lambda).

```kotlin
inline fun process(
    crossinline action: () -> Unit,
    noinline onError: (Throwable) -> Unit
) {
    try {
        action() // re-invoked later or in another context → crossinline needed
    } catch (t: Throwable) {
        onError(t) // kept as a normal function value → noinline
    }
}
```

Reified type parameters

Normally, generics are erased on the JVM. Inline functions can use reified type parameters, letting you access the actual type at runtime.

```kotlin
inline fun <reified T> castOrNull(any: Any?): T? =
    if (any is T) any else null

val s: String? = castOrNull("hello") // works thanks to reified T
```

When to use inline functions

- Tiny utility functions called frequently in hot paths
- Higher-order functions that would otherwise allocate lambdas
- APIs that benefit from reified type parameters (e.g., JSON parsing helpers, reflection-light utilities)

When to avoid

- Large functions: inlining duplicates code at call sites and can increase bytecode size
- Functions rarely called or not performance critical
- Public API functions where binary size and inlining across module boundaries should be considered

---

#### Value Classes: What and Why

A value class wraps a single value to provide type safety with minimal or zero runtime overhead. On supported targets, the compiler can represent the value class as its underlying value at runtime, avoiding extra allocation.

```kotlin
@JvmInline
value class UserId(val value: String) {
    init { require(value.isNotBlank()) { "UserId cannot be blank" } }
}

@JvmInline
value class Cents(val value: Long) {
    operator fun plus(other: Cents) = Cents(this.value + other.value)
    operator fun minus(other: Cents) = Cents(this.value - other.value)
}

fun pay(userId: UserId, amount: Cents) { /* ... */ }
```

Benefits

- Type safety: prevent mixing parameters with the same primitive type (e.g., UserId vs. OrderId)
- Domain modeling: make illegal states unrepresentable (e.g., NonEmptyString)
- Performance: often no extra allocation compared to using the raw primitive

Common domain wrappers

```kotlin
@JvmInline value class OrderId(val value: String)
@JvmInline value class Email(val value: String)
@JvmInline value class Percentage private constructor(val value: Int) {
    companion object { fun of(raw: Int) = Percentage(raw.coerceIn(0, 100)) }
}
```

Pitfalls and edge cases

- Boxing can occur with generics, interfaces, and when the type is nullable (UserId?).
- Be careful with reflection and serialization; you may need custom adapters.
- Do not abuse value classes as data classes: they only hold one property.

---

#### Practical Use Cases

1) Safer IDs in APIs

```kotlin
@JvmInline value class ProductId(val value: String)
@JvmInline value class UserId(val value: String)

fun linkUserToProduct(userId: UserId, productId: ProductId) { /* ... */ }
```

2) Money and units

```kotlin
@JvmInline value class Money(val cents: Long) {
    operator fun plus(other: Money) = Money(cents + other.cents)
}

@JvmInline value class Meters(val value: Double)
```

3) Serialization helpers with reified inline

```kotlin
inline fun <reified T> parse(json: String): T {
    // Usually delegated to your JSON library using T::class
    // Placeholder implementation:
    error("Provide adapter for ${T::class}")
}
```

4) Allocation-free functional utilities

```kotlin
inline fun <T> T.alsoIf(condition: Boolean, crossinline block: (T) -> Unit): T {
    if (condition) block(this)
    return this
}
```

---

#### Performance Notes

- JVM: Value classes are typically optimized away; boxing appears when using generics, interfaces, or nullable forms. Measure in your context.
- Kotlin/Native: Value classes are represented efficiently; inlining helps reduce call overhead.
- JS: Representation differs; rely on benchmarks to validate assumptions.
- Inlining increases bytecode size; avoid inlining large bodies used in many places.

---

#### Interoperability

- JVM and Android: Use @JvmInline for best interop; beware of Java callers seeing the underlying type.
- Multiplatform: Value classes and inline functions are supported on common code; check platform-specific behavior for boxing.
- Libraries: If exposing value classes publicly, document serialization and interop story (e.g., how JSON adapters treat them).

---

#### Testing Tips

- Write unit tests around your value class invariants (constructors, operators).
- Benchmark hot paths when introducing inline functions; confirm gains with your workload.
- For serialization, add round-trip tests for value classes.

---

#### When Not to Use

- Value classes that hide complex state (more than one property) → consider data classes instead.
- Inline functions with large bodies or rarely used code.
- Public APIs where callers from other languages (Java, Swift) might be confused by the wrapper type.

---

#### Summary

- Inline functions remove indirection and unlock reified generics.
- Value classes improve type safety with minimal runtime cost.
- Together, they enable expressive, safe, and efficient Kotlin APIs.
