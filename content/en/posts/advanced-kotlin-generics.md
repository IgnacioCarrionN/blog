---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Advanced Generics and Variance in Kotlin: A Comprehensive Guide"
date: 2025-03-21T08:00:00+01:00
description: "Master Kotlin's advanced generic types and variance concepts with practical examples and real-world applications"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/advanced-generics.png
draft: false
tags:
- kotlin
- generics
- variance
- type-safety
---

# Advanced Generics and Variance in Kotlin: A Comprehensive Guide

Understanding advanced generics and variance in Kotlin is crucial for writing type-safe, reusable code. This article explores these concepts in depth, providing practical examples and real-world applications.

## Understanding Variance

Variance in Kotlin determines how generic types with different type arguments relate to each other. Understanding variance is easier when thinking in terms of producers and consumers:

- **Producer**: Only produces/provides values of type T (output)
- **Consumer**: Only consumes/accepts values of type T (input)

This producer/consumer relationship directly maps to the two types of variance:

- **Covariance** (`out`): Used for producers - only outputs values
- **Contravariance** (`in`): Used for consumers - only inputs values

Here's how producers and consumers work with types:

```
Type Hierarchy:    Producer<T>             Consumer<T>
Any               ▲ can produce            ▼ can consume
  └── Number      │ more specific         │ more general
       └── Int    │ types                 │ types
```

### For producers (covariant, `out`):
```kotlin
interface Producer<out T> {
    fun get(): T              // OK: Producing/returning T
    // fun set(item: T) {}    // Error: Can't consume T
}

// Simple producer implementation
class IntProducer(private val value: Int) : Producer<Int> {
    override fun get(): Int = value
}

// Type safety with covariance:
// 1. IntProducer.get() returns Int
// 2. Int is a subtype of Number
// 3. Therefore, it's safe to use Producer<Int> as Producer<Number>
val intProducer: Producer<Int> = IntProducer(42)
val numberProducer: Producer<Number> = intProducer  // Safe: Int is always a Number

// Without 'out' modifier, this wouldn't compile:
// class RegularBox<T>(val item: T)  // invariant
// val intBox: RegularBox<Int> = RegularBox(42)
// val numBox: RegularBox<Number> = intBox  // Error: Type mismatch

// We can use the numberProducer wherever a Number is expected
fun printNumber(producer: Producer<Number>) {
    println("Number: ${producer.get()}")  // Safe: we know we'll get a Number
}

printNumber(intProducer)  // Works because Producer is covariant (out)
```

### For consumers (contravariant, `in`):
```kotlin
interface Consumer<in T> {
    fun set(item: T)         // OK: Consuming/accepting T
    // fun get(): T {}       // Error: Can't produce T
}

// Simple consumer implementation
class NumberProcessor : Consumer<Number> {
    override fun set(item: Number) {
        println("Processing number: ${item.toDouble()}")
    }
}

// Type safety with contravariance:
// 1. NumberProcessor.set() accepts any Number
// 2. Int is a subtype of Number
// 3. Therefore, it's safe to use Consumer<Number> as Consumer<Int>
val numberConsumer: Consumer<Number> = NumberProcessor()
val intConsumer: Consumer<Int> = numberConsumer     // Safe: anything that can handle Number can handle Int

// Without 'in' modifier, this wouldn't compile:
// class Processor<T>(val process: (T) -> Unit)  // invariant
// val numProcessor: Processor<Number> = Processor { println(it) }
// val intProcessor: Processor<Int> = numProcessor  // Error: Type mismatch

// We can use the intConsumer with Int values
fun processInt(consumer: Consumer<Int>) {
    consumer.set(42)  // Safe: we know the consumer can handle any Number, including Int
}

processInt(numberConsumer)  // Works because Consumer is contravariant (in)
```

The restrictions make sense because:
- A producer of Ints can safely produce them where Numbers are needed (every Int is a Number)
- A consumer of Numbers can safely consume Ints (it knows how to handle any Number)

Here's a simple way to remember it:
- If a class only produces/returns T, make it covariant with `out T` (can use more specific types)
- If a class only consumes/accepts T, make it contravariant with `in T` (can use more general types)
- If a class both produces and consumes T, it should remain invariant (type must match exactly)

### Invariance (Default)

By default, generic types in Kotlin are invariant, meaning there's no subtype relationship between different instantiations of the generic type.

```kotlin
class Box<T>(var value: T)

fun main() {
    val stringBox = Box("Hello")
    // val anyBox: Box<Any> = stringBox // This won't compile
}
```

## Declaration-site vs Use-site Variance

Kotlin supports two ways to specify variance: declaration-site variance (using `in` or `out` on the class/interface declaration) and use-site variance (using type projections). Each approach has its own use cases and benefits.

### Declaration-site Variance

Declaration-site variance is specified at the type parameter declaration of a class or interface. This approach is preferred when a class can only use the type parameter in one way throughout its entire implementation.

```kotlin
// Declaration-site variance example
interface Producer<out T> {
    fun produce(): T              // Can only produce/return T
    // fun consume(item: T) {}    // Error: Can't consume T in an out position
}

interface Consumer<in T> {
    fun consume(item: T)          // Can only consume T
    // fun produce(): T {}        // Error: Can't produce T in an in position
}

// Usage is straightforward - variance is handled automatically
class StringProducer : Producer<String> {
    override fun produce(): String = "Hello"
}

val producer: Producer<Any> = StringProducer() // OK: String is more specific than Any
```

### Use-site Variance

Use-site variance (also known as type projection) is specified at the point of usage. This is useful when a type can be used both as a producer and consumer, but in a specific usage, you want to restrict it to one role.

```kotlin
// Class with invariant type parameter
class Box<T>(var value: T) {
    fun get(): T = value
    fun set(value: T) { this.value = value }
}

// Use-site variance examples
fun copyOut(from: Box<out Number>, to: MutableList<Number>) {
    // 'from' is projected to be covariant (producer)
    // Can only call methods that return Number
    to.add(from.get())  // OK
    // from.set(42)     // Error: Can't call set on projected type
}

fun copyIn(to: Box<in Number>, from: List<Int>) {
    // 'to' is projected to be contravariant (consumer)
    // Can only call methods that accept Number
    to.set(from.first())  // OK
    // val x: Number = to.get()  // Error: Return type is projected to Nothing
}
```

### When to Use Each Approach

1. Use Declaration-site Variance When:
   - The class can only use the type parameter in one way (only produce or only consume)
   - You want to enforce the usage pattern across all usages of the class
   - The API design is clear about its variance requirements

2. Use Use-site Variance When:
   - The class needs to both produce and consume the type in general
   - You want to restrict variance at specific usage points
   - You need flexibility in how the type is used in different contexts

## Generic Constraints

Kotlin allows you to specify upper bounds for type parameters, restricting what types can be used.

```kotlin
interface Drawable {
    fun draw()
}

class Canvas<T : Drawable> {
    fun drawAll(elements: List<T>) {
        elements.forEach { it.draw() }
    }
}

class Circle : Drawable {
    override fun draw() = println("Drawing Circle")
}

val canvas = Canvas<Circle>()
canvas.drawAll(listOf(Circle(), Circle()))
```

### Multiple Constraints

You can specify multiple constraints using where clause:

```kotlin
fun <T> copyWhenBothValid(
    source: T,
    destination: T
) where T : Drawable,
        T : Comparable<T> {
    if (source > destination) {
        destination.draw()
    }
}
```

## Best Practices and Guidelines

1. Use `out` when your class only produces values of type T
2. Use `in` when your class only consumes values of type T
3. Use invariance when your class both produces and consumes values of type T
4. Prefer declaration-site variance (`out`/`in` on the class) over use-site variance when possible

## Conclusion

Advanced generics and variance in Kotlin provide powerful tools for building type-safe, reusable abstractions. By understanding these concepts and applying them appropriately, you can write more robust and maintainable code. Remember to:

- Use variance modifiers (`out`/`in`) when appropriate
- Apply generic constraints to ensure type safety
- Consider both declaration-site and use-site variance
- Be mindful of type erasure and nullability

The proper use of these features leads to more elegant and safer code, reducing the likelihood of runtime errors and making your codebase more maintainable.
