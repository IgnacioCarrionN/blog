---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Understanding SOLID Principles with Kotlin Examples"
date: 2025-01-08T08:00:00+01:00
description: "SOLID Principles explained with Kotlin Examples."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/di-fix.png
draft: false
tags: 
- kotlin
- solid
- architecture
---

### Understanding SOLID Principles with Kotlin Examples

The SOLID principles are a set of design principles that make software designs more understandable, flexible, and maintainable. Introduced by Robert C. Martin, these principles are a cornerstone of object-oriented programming and are especially relevant when building complex systems. In this blog post, we’ll explore each principle with examples written in Kotlin, a language that brings modern syntax and powerful features to the table.

---

#### 1. Single Responsibility Principle (SRP)
**A class should have one, and only one, reason to change.**

This principle ensures that a class has a single responsibility, making it easier to maintain and less prone to bugs.

**Breaking SRP:**

```kotlin
class ReportManager {
    fun generateReport(data: String): String {
        // Logic to generate report
        return "Report: $data"
    }

    fun saveReport(report: String) {
        // Logic to save report
        println("Report saved: $report")
    }
}
```

In this example, the `ReportManager` class violates SRP because it has two responsibilities: generating and saving reports. Any change in report generation logic or saving logic would require modifying the same class.

**Fixing SRP:**

```kotlin
class ReportGenerator {
    fun generateReport(data: String): String {
        // Logic to generate report
        return "Report: $data"
    }
}

class ReportSaver {
    fun saveReport(report: String) {
        // Logic to save report
        println("Report saved: $report")
    }
}

fun main() {
    val generator = ReportGenerator()
    val saver = ReportSaver()

    val report = generator.generateReport("Sales Data")
    saver.saveReport(report)
}
```
By separating responsibilities, we make each class focused and easier to test independently.

---

#### 2. Open/Closed Principle (OCP)
**Software entities should be open for extension but closed for modification.**

You can add new functionality by extending classes without changing the existing code.

**Breaking OCP:**

```kotlin
class Discount {
    fun calculate(price: Double, type: String): Double {
        return when (type) {
            "none" -> price
            "percentage" -> price * 0.9
            else -> throw IllegalArgumentException("Unknown discount type")
        }
    }
}
```

Here, adding a new discount type requires modifying the `calculate` method, which violates OCP.

**Fixing OCP:**

```kotlin
interface DiscountStrategy {
    fun calculate(price: Double): Double
}

class NoDiscount : DiscountStrategy {
    override fun calculate(price: Double): Double = price
}

class PercentageDiscount(private val percentage: Double) : DiscountStrategy {
    override fun calculate(price: Double): Double = price * (1 - percentage / 100)
}

class DiscountCalculator(private val strategy: DiscountStrategy) {
    fun calculate(price: Double): Double = strategy.calculate(price)
}

fun main() {
    val noDiscount = DiscountCalculator(NoDiscount())
    println("Price after no discount: ${noDiscount.calculate(100.0)}")

    val percentageDiscount = DiscountCalculator(PercentageDiscount(10.0))
    println("Price after 10% discount: ${percentageDiscount.calculate(100.0)}")
}
```

By using interfaces and composition, we achieve a design that is open to extension (new discount strategies) and closed to modification (no changes to existing classes).

---

#### 3. Liskov Substitution Principle (LSP)
**Objects of a superclass should be replaceable with objects of a subclass without affecting the correctness of the program.**

This principle ensures that derived classes honor the expectations set by their base class.

**Breaking LSP:**

```kotlin
open class Bird {
    open fun fly() {
        println("Flying")
    }
}

class Sparrow : Bird()

class Penguin : Bird() {
    override fun fly() {
        throw UnsupportedOperationException("Penguins can't fly")
    }
}

fun main() {
    val birds: List<Bird> = listOf(Sparrow(), Penguin())

    for (bird in birds) {
        bird.fly() // This will fail for Penguin
    }
}
```

In this example, Penguin violates LSP because it cannot fulfill the contract of `Bird`. A better approach is to refactor the design:

```kotlin
interface Flyable {
    fun fly()
}

class Sparrow : Flyable {
    override fun fly() {
        println("Flying")
    }
}

class Penguin {
    fun swim() {
        println("Swimming")
    }
}
```
Now, behaviors are segregated, and LSP is upheld.

---

#### 4. Interface Segregation Principle (ISP)
**Clients should not be forced to depend on methods they do not use.**

This principle promotes creating specific interfaces rather than a bloated one.

**Breaking ISP:**

```kotlin
interface Machine {
    fun print()
    fun scan()
    fun fax()
}

class OldPrinter : Machine {
    override fun print() {
        println("Printing")
    }

    override fun scan() {
        throw UnsupportedOperationException("Scan not supported")
    }

    override fun fax() {
        throw UnsupportedOperationException("Fax not supported")
    }
}
```

This implementation forces `OldPrinter` to implement methods it doesn’t support, violating ISP.

**Fixing ISP:**

```kotlin
interface Printer {
    fun print()
}

interface Scanner {
    fun scan()
}

class SimplePrinter : Printer {
    override fun print() {
        println("Printing")
    }
}
```
By splitting the functionalities into separate interfaces, we allow devices to implement only what they need.

---

#### 5. Dependency Inversion Principle (DIP)
**High-level modules should not depend on low-level modules. Both should depend on abstractions.**

This principle reduces the coupling between high-level and low-level modules by introducing abstractions.

**Breaking DIP:**

```kotlin
class EmailService {
    fun sendEmail(message: String) {
        println("Sending Email: $message")
    }
}

class NotificationSender {
    private val emailService = EmailService()

    fun notifyUser(message: String) {
        emailService.sendEmail(message)
    }
}
```

Here, `NotificationSender` is tightly coupled to `EmailService`, making it difficult to switch to a different notification service.

**Fixing DIP:**

```kotlin
interface NotificationService {
    fun sendNotification(message: String)
}

class EmailService : NotificationService {
    override fun sendNotification(message: String) {
        println("Sending Email: $message")
    }
}

class SMSService : NotificationService {
    override fun sendNotification(message: String) {
        println("Sending SMS: $message")
    }
}

class NotificationSender(private val service: NotificationService) {
    fun notifyUser(message: String) {
        service.sendNotification(message)
    }
}

fun main() {
    val emailSender = NotificationSender(EmailService())
    emailSender.notifyUser("Hello via Email")

    val smsSender = NotificationSender(SMSService())
    smsSender.notifyUser("Hello via SMS")
}
```
Here, `NotificationSender` depends on the `NotificationService` abstraction, making it flexible to work with any notification type.

---

### Conclusion
The SOLID principles form the foundation for building robust and scalable software. Kotlin, with its expressive syntax and modern features, allows developers to implement these principles elegantly. By adhering to these principles, you can create code that is easier to maintain, extend, and adapt to changing requirements.
