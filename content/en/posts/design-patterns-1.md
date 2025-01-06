---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Kotlin Design Patterns - Part 1"
date: 2024-12-30T08:00:00+01:00
description: "Kotlin Design Patterns - Part 1"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/singleton-pattern.png
draft: false
tags: 
- kotlin
- design-patterns
- architecture
---

### **Exploring Design Patterns in Kotlin - Part 1**

#### Design Patterns Series

- [Part 2](https://carrion.dev/en/posts/design-patterns-2/)

---

Design patterns are proven solutions to common problems in software design. With Kotlin’s expressive syntax and modern features, implementing these patterns often becomes cleaner and more concise. In this post, we’ll explore **Singleton**, **Factory Method**, **Builder**, **Adapter** and **Decorator** patterns, delving into their purpose, use cases, and Kotlin implementations.

---

#### **1. Singleton Pattern**

The **Singleton Pattern** ensures that a class has only one instance and provides a global access point to it.

##### **When to Use**

- Managing shared resources like database connections, logging, or configuration settings.

##### **Kotlin Implementation**

Kotlin’s `object` keyword provides a straightforward way to create a Singleton.

```kotlin
object DatabaseConnection {
    fun connect() {
        println("Connecting to database...")
    }
}
```

##### **Usage**

```kotlin
fun main() {
    DatabaseConnection.connect()
}
```

##### **Advantages in Kotlin**

- Thread-safe by default.
- Requires minimal boilerplate compared to traditional implementations in other languages.

---

#### **2. Factory Method Pattern**

The **Factory Method Pattern** delegates the creation of objects to subclasses or helper functions, providing flexibility in object instantiation.

##### **When to Use**

- When creating objects involves logic or complexity.
- To decouple object creation from client code.

##### **Kotlin Implementation**

```kotlin
interface Shape {
    fun draw()
}

class Circle : Shape {
    override fun draw() = println("Drawing a Circle")
}

class Rectangle : Shape {
    override fun draw() = println("Drawing a Rectangle")
}

object ShapeFactory {
    fun createShape(type: String): Shape = when (type) {
        "Circle" -> Circle()
        "Rectangle" -> Rectangle()
        else -> throw IllegalArgumentException("Unknown shape type")
    }
}
```

##### **Usage**

```kotlin
fun main() {
    val shape = ShapeFactory.createShape("Circle")
    shape.draw()
}
```

---

#### **3. Builder Pattern**

The **Builder Pattern** is used to construct complex objects step by step. It’s especially useful when an object has many optional parameters or configurations.

##### **When to Use**

- To avoid constructors with numerous parameters.
- When the construction process is complex or involves multiple steps.

##### **Kotlin Implementation**

Kotlin’s `apply` and `DSL` capabilities simplify the Builder Pattern.

```kotlin
class Car(val make: String, val model: String, val year: Int) {
    class Builder {
        private var make = ""
        private var model = ""
        private var year = 0

        fun make(make: String) = apply { this.make = make }
        fun model(model: String) = apply { this.model = model }
        fun year(year: Int) = apply { this.year = year }

        fun build() = Car(make, model, year)
    }
}
```

##### **Usage**

```kotlin
fun main() {
    val car = Car.Builder()
        .make("Toyota")
        .model("Corolla")
        .year(2022)
        .build()

    println("${car.make} ${car.model}, ${car.year}")
}
```

##### **Why Kotlin?**

Chaining methods with `apply` allows a concise and expressive syntax for constructing objects.

---

#### **4. Adapter Pattern**

The **Adapter Pattern** is used to bridge the gap between incompatible interfaces by translating one interface to another.

##### **When to Use**

- Integrating with legacy code or external libraries.
- When two systems or components need to work together but have incompatible interfaces.

##### **Kotlin Implementation**

```kotlin
// Existing integer provider interface
interface OldProvider {
    fun provide(): Int
}

class RandomIntProvider : OldProvider {
    override fun provide(): Int = (1..100).random()
}

// Target string provider interface
interface NewProvider {
    fun provide(): String
}

// Adapter class
class OldToNewProviderAdapter(private val intProvider: OldProvider) : NewProvider {
    override fun provide(): String = "Provided number: ${intProvider.provide()}"
}
```

##### **Usage**

```kotlin
fun main() {
    val intProvider = RandomIntProvider()
    val stringProvider: NewProvider = OldToNewProviderAdapter(intProvider)

    println(stringProvider.provideString())
}
```

##### **Why Kotlin?**

Kotlin’s primary constructors and concise class syntax simplify the implementation of wrapper classes.

---

#### **5. Decorator Pattern**

The **Decorator Pattern** dynamically adds behavior to objects without altering their structure.

##### **When to Use**

- To extend functionality of a class at runtime.
- When subclassing would lead to a bloated hierarchy.

##### **Kotlin Implementation**

```kotlin
interface Coffee {
    fun cost(): Double
    fun description(): String
}

class SimpleCoffee : Coffee {
    override fun cost() = 5.0
    override fun description() = "Simple Coffee"
}

class MilkDecorator(private val coffee: Coffee) : Coffee {
    override fun cost() = coffee.cost() + 1.5
    override fun description() = coffee.description() + ", Milk"
}
```

##### **Usage**

```kotlin
fun main() {
    val coffee = SimpleCoffee()
    val coffeeWithMilk = MilkDecorator(coffee)

    println("${coffeeWithMilk.description()} costs \$${coffeeWithMilk.cost()}")
}
```

---

### **Conclusion**

Kotlin’s modern features like `object`, `when`, and `apply` make implementing traditional design patterns easier and more expressive. These patterns not only solve common design challenges but also demonstrate how Kotlin enhances their implementation.

Are there other patterns you’d like me to cover in future posts?
