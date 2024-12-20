---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Kotlin contracts"
date: 2024-12-20T08:00:00+01:00
description: "Advanced Kotlin - Contracts"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/kotlin-contract.png
draft: false
tags: 
- kotlin
- android
- advanced
---

### **Mastering Kotlin Contracts: Unlocking Smarter Code Analysis**

Kotlin never ceases to amaze with its features that combine elegance and power. One advanced yet often underutilized tool in Kotlin's arsenal is **Contracts**. Contracts let you guide the Kotlin compiler to make smarter decisions about your code—resulting in better null safety, optimized performance, and fewer runtime errors.

---

### **What Are Kotlin Contracts?**

Kotlin Contracts allow you to define **rules** about how your functions behave, helping the compiler perform advanced static analysis. They enable features like smart-casts and context-aware checks beyond Kotlin’s default capabilities.

---

### **Why Use Contracts?**

1. **Improve Null Safety**: Eliminate redundant null checks by telling the compiler when something is guaranteed to be non-null.
2. **Optimize Smart-Casts**: Make the compiler aware of variable types in custom scenarios.
3. **Reduce Boilerplate**: Write cleaner, more intuitive code by offloading repetitive checks to the compiler.

---

### **Examples of Kotlin Contracts in Action**

#### 1. **Simplify Null Checks**

Let’s create a custom utility to validate non-null values:

```kotlin
@OptIn(ExperimentalContracts::class)
inline fun <T> requireNotNull(value: T?, message: String): T {
    contract {
        returns() implies (value != null)
    }
    if (value == null) {
        throw IllegalArgumentException(message)
    }
    return value
}

fun processName(name: String?) {
    val nonNullName = requireNotNull(name, "Name cannot be null")
    // No need for additional null checks; compiler knows 'nonNullName' is not null!
    println("Processing name: $nonNullName")
}

fun main() {
    processName("John")  // Works fine
    // processName(null) // Throws an IllegalArgumentException
}
```

Something similar is implemented in the functions `require` and `requireNotNull` from the Kotlin standard lib:

```kotlin
/**
 * Throws an [IllegalArgumentException] with the result of calling [lazyMessage] if the [value] is false.
 *
 * @sample samples.misc.Preconditions.failRequireWithLazyMessage
 */
@kotlin.internal.InlineOnly
public inline fun require(value: Boolean, lazyMessage: () -> Any): Unit {
    contract {
        returns() implies value
    }
    if (!value) {
        val message = lazyMessage()
        throw IllegalArgumentException(message.toString())
    }
}

/**
 * Throws an [IllegalArgumentException] with the result of calling [lazyMessage] if the [value] is null. Otherwise
 * returns the not null value.
 *
 * @sample samples.misc.Preconditions.failRequireNotNullWithLazyMessage
 */
@kotlin.internal.InlineOnly
public inline fun <T : Any> requireNotNull(value: T?, lazyMessage: () -> Any): T {
    contract {
        returns() implies (value != null)
    }

    if (value == null) {
        val message = lazyMessage()
        throw IllegalArgumentException(message.toString())
    } else {
        return value
    }
}
```

##### **How Contracts Help Here**

- The **`returns() implies (value != null)`** contract tells the compiler:
  > If the function returns successfully, then `value` is guaranteed to be non-null.
- This enables smart-casts, so you don’t need manual null checks after the function call.

---

#### 2. **Custom Assertions**

Here’s how contracts can be used to define custom assertion functions:

```kotlin
@OptIn(ExperimentalContracts::class)
fun assertValidState(condition: Boolean, message: String) {
    contract {
        returns() implies condition
    }
    if (!condition) {
        throw IllegalStateException(message)
    }
}

fun performOperation(state: Boolean) {
    val state: Any? = "Hello"
    assertValidState(state is String, "Is String")
    // Here the compiler knows that the state val is of type String so no need to other cast checks
    println("String length: ${assertion.length}")
}

fun main() {
    performOperation(true)   // Prints success
    // performOperation(false) // Throws IllegalStateException
}
```

---

#### 3. **Smart-Casts with Custom Conditions**

Let’s create a utility function that checks if a value matches a specific type. This will demonstrate how contracts enable smarter casting:

```kotlin
@OptIn(ExperimentalContracts::class)
inline fun <reified T> isOfType(value: Any?): Boolean {
    contract {
        returns(true) implies (value is T)
    }
    return value is T
}

fun main() {
    val input: Any? = "Hello, Kotlin!"

    if (isOfType<String>(input)) {
        println("String length: ${input.length}")
    }

    val inputInt: Any? = 10
    if (isOfType<Int>(inputInt)) {
        println("The value is an integer ${input.toUInt()}")
    }
}
```

With this implementation, the compiler knows that within the `if` block, `input` is a `String`, thanks to the contract defined in `isOfType`. Also the compilers knows that `inputInt` is an `Int` so you don't need to cast it.

---

#### 4. **Optimizing Flow Control**

Contracts can simplify flow control by enabling the compiler to understand loop invariants or conditions. Here’s an example:

```kotlin
inline fun isNotEmpty(list: List<*>?): Boolean {
    contract {
        returns(true) implies (list != null && list.isNotEmpty())
    }
    return list != null && list.isNotEmpty()
}

fun processItems(items: List<String>?) {
    if (isNotEmpty(items)) {
        // Compiler knows items is non-null and not empty
        println("Processing ${items.size} items")
    } else {
        println("No items to process")
    }
}

fun main() {
    processItems(listOf("A", "B", "C"))
    processItems(null)
    processItems(emptyList())
}
```

##### **Output**

```
Processing 3 items
No items to process
No items to process
```

---

### **When to Use Contracts**

Contracts are ideal for:

1. **Library Development**: Safeguard public APIs by enforcing preconditions.
2. **DSLs and Frameworks**: Simplify type-checking and state validations in Kotlin DSLs.
3. **Performance Optimization**: Reduce runtime checks by letting the compiler infer conditions at compile time.

---

### **Conclusion**

Kotlin Contracts are a hidden gem that can elevate your code by improving safety, reducing boilerplate, and enabling smarter compiler analysis. Whether you're building libraries, writing complex DSLs, or just optimizing everyday code, contracts provide a powerful tool to guide the Kotlin compiler and ensure code correctness.

Also keep in mind that contracts are annotated as experimental feature but they are in Kotlin since 1.3 version and are being used in the standard library so they are stable enough to use them.