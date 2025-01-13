---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Kotlin Design Patterns - Part 2"
date: 2025-01-06T08:00:00+01:00
description: "Kotlin Design Patterns - Part 2"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/proxy-pattern.png
draft: false
tags: 
- kotlin
- design-patterns
- architecture
---

### **Exploring Design Patterns in Kotlin: Part 2**

#### Design Patterns Series

- [Part 1](https://carrion.dev/en/posts/design-patterns-1/)
- [Part 2](https://carrion.dev/en/posts/design-patterns-2/)
- [Part 3](https://carrion.dev/en/posts/design-patterns-3/)

---

After the overwhelming response to our first post on [Kotlin design patterns](https://carrion.dev/en/posts/design-patterns-1/), we’re back with more! In this second part, we’ll dive into **Prototype**, **Composite**, **Proxy**, **Observer**, and **Strategy** patterns. These patterns solve a variety of design challenges and demonstrate Kotlin’s expressive capabilities.

---

#### **1. Prototype Pattern**

The **Prototype Pattern** is used to create new objects by copying an existing object, ensuring efficient object creation.

##### **When to Use**

- When creating a new instance is costly or complex.
- To avoid creating instances of subclasses repeatedly.

##### **Kotlin Implementation**

Using Kotlin’s `data` class and its built-in `copy` function simplifies this pattern.

```kotlin
data class Document(var title: String, var content: String, var author: String)

fun main() {
    val original = Document("Design Patterns", "Content about patterns", "John Doe")
    val copy = original.copy(title = "Prototype Pattern")

    println("Original: $original")
    println("Copy: $copy")
}
```

##### **Why Kotlin?**

Kotlin’s `data` classes inherently support copying with minimal boilerplate, making the Prototype Pattern a breeze to implement.

---

#### **2. Composite Pattern**

The **Composite Pattern** is used to treat individual objects and groups of objects uniformly.

##### **When to Use**

- When you have a tree structure and want to manipulate it in a consistent way.

##### **Kotlin Implementation**

```kotlin
interface Logger {
    fun log(message: String)
}

class ConsoleLogger : Logger {
    override fun log(message: String) {
        println(message)
    }
}

class FileLogger(private val filePath: String) : Logger {
    override fun log(message: String) {
        // Implementation for writing logs to a file
    }
}

class RootLogger(private val loggers: List<Logger>) : Logger {
    override fun log(message: String) {
        loggers.forEach { it.log(message) }
    }
}

fun main() {
    val consoleLogger = ConsoleLogger()
    val fileLogger = FileLogger("/path/to/log.txt")
    val rootLogger = RootLogger(listOf(consoleLogger, fileLogger))

    rootLogger.log("Composite Pattern Example")
}
```

---

#### **3. Proxy Pattern**

The **Proxy Pattern** provides a surrogate or placeholder to control access to another object.

##### **When to Use**

- To control access to a resource.
- To add functionality without modifying the actual object.

##### **Kotlin Implementation**

```kotlin
interface Service {
    fun fetchData(): String
}

class RealService : Service {
    override fun fetchData() = "Data from Real Service"
}

class ProxyService(private val realService: RealService) : Service {
    override fun fetchData(): String {
        println("Proxy: Checking access before delegating.")
        return realService.fetchData()
    }
}

fun main() {
    val proxy = ProxyService(RealService())
    println(proxy.fetchData())
}
```

---

#### **4. Observer Pattern**

The **Observer Pattern** defines a one-to-many dependency, so when one object changes state, all its dependents are notified.

##### **When to Use**

- For event-driven systems.
- When multiple components need to react to state changes.

##### **Kotlin Implementation**

Using Kotlin's `fun interface` makes defining listeners more concise.

```kotlin
fun interface StateChangeListener {
    fun onStateChanged(oldState: String, newState: String)
}

class Subject {
    private val listeners = mutableListOf<StateChangeListener>()

    var state: String by Delegates.observable("Initial State") { _, old, new ->
        listeners.forEach { it.onStateChanged(old, new) }
    }

    fun addListener(listener: StateChangeListener) {
        listeners.add(listener)
    }
}

fun main() {
    val subject = Subject()

    subject.addListener { oldState, newState ->
        println("Listener 1: State changed from '$oldState' to '$newState'")
    }

    subject.addListener { oldState, newState ->
        println("Listener 2: State changed from '$oldState' to '$newState'")
    }

    subject.state = "State 1"
    subject.state = "State 2"
}
```

##### **Why Kotlin?**

Using `fun interface` simplifies the implementation of single-method interfaces and reduces boilerplate for listeners. Additionally, Kotlin's `Delegates.observable` makes observing state changes straightforward and powerful, further enhancing the implementation of the Observer Pattern.

---

#### **5. Strategy Pattern**

The **Strategy Pattern** defines a family of algorithms, encapsulates each one, and makes them interchangeable.

##### **When to Use**

- When you need multiple algorithms for a specific task.
- To avoid hardcoding algorithm logic.

##### **Kotlin Implementation**

```kotlin
interface PaymentStrategy {
    fun pay(amount: Double)
}

class CreditCardPayment : PaymentStrategy {
    override fun pay(amount: Double) = println("Paid $$amount using Credit Card.")
}

class PayPalPayment : PaymentStrategy {
    override fun pay(amount: Double) = println("Paid $$amount using PayPal.")
}

class PaymentContext(private var strategy: PaymentStrategy) {
    fun setStrategy(strategy: PaymentStrategy) {
        this.strategy = strategy
    }

    fun executePayment(amount: Double) = strategy.pay(amount)
}

fun main() {
    val context = PaymentContext(CreditCardPayment())
    context.executePayment(100.0)

    context.setStrategy(PayPalPayment())
    context.executePayment(200.0)
}
```

---

### **Conclusion**

With Kotlin, design patterns like **Prototype**, **Composite**, **Proxy**, **Observer**, and **Strategy** become more intuitive and powerful. These patterns are not just tools—they're stepping stones to cleaner and more maintainable code.
