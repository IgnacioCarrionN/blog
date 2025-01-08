---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Entendiendo los principios SOLID con ejemplos en Kotlin"
date: 2025-01-08T08:00:00+01:00
description: "Principios SOLID explicados con ejemplos de Kotlin."
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

### Entendiendo los principios SOLID con ejemplos en Kotlin

Los principios SOLID son un conjunto de principios de diseño que hacen que los diseños de software sean más comprensibles, flexibles y mantenibles. Introducidos por Robert C. Martin, estos principios son una piedra angular de la programación orientada a objetos y son especialmente relevantes al construir sistemas complejos. En este blog, exploraremos cada principio con ejemplos escritos en Kotlin, un lenguaje que ofrece una sintaxis moderna y características poderosas.

---

#### 1. Principio de Responsabilidad Única (SRP)
**Una clase debe tener una, y solo una, razón para cambiar.**

Este principio asegura que una clase tenga una única responsabilidad, lo que la hace más fácil de mantener y menos propensa a errores.

**Rompiendo SRP:**

```kotlin
class ReportManager {
    fun generateReport(data: String): String {
        // Lógica para generar reporte
        return "Report: $data"
    }

    fun saveReport(report: String) {
        // Lógica para guardar reporte
        println("Report saved: $report")
    }
}
```

En este ejemplo, la clase `ReportManager` viola el SRP porque tiene dos responsabilidades: generar y guardar reportes. Cualquier cambio en la lógica de generación o de guardado requeriría modificar la misma clase.

**Corrigiendo SRP:**

```kotlin
class ReportGenerator {
    fun generateReport(data: String): String {
        // Lógica para generar reporte
        return "Report: $data"
    }
}

class ReportSaver {
    fun saveReport(report: String) {
        // Lógica para guardar reporte
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
Separando responsabilidades, hacemos que cada clase esté enfocada y sea más fácil de probar de manera independiente.

---

#### 2. Principio Abierto/Cerrado (OCP)
**Las entidades de software deben estar abiertas para extensión, pero cerradas para modificación.**

Puedes añadir nueva funcionalidad extendiendo clases sin cambiar el código existente.

**Rompiendo OCP:**

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

Aquí, añadir un nuevo tipo de descuento requiere modificar el método `calculate`, lo que viola el OCP.

**Corrigiendo OCP:**

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

Usando interfaces y composición, logramos un diseño que está abierto a la extensión (nuevas estrategias de descuento) y cerrado a la modificación (sin cambios en las clases existentes).

---

#### 3. Principio de Sustitución de Liskov (LSP)
**Los objetos de una superclase deben poder ser reemplazados con objetos de una subclase sin afectar la corrección del programa.**

Este principio asegura que las clases derivadas respeten las expectativas establecidas por su clase base.

**Rompiendo LSP:**

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
        bird.fly() // Esto fallará para Penguin
    }
}
```

En este ejemplo, Penguin viola LSP porque no puede cumplir el contrato de `Bird`. Una mejor aproximación es refactorizar el diseño:

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
Ahora, los comportamientos están segregados, y se respeta el LSP.

---

#### 4. Principio de Segregación de Interfaces (ISP)
**Los clientes no deberían estar obligados a depender de métodos que no utilizan.**

Este principio promueve la creación de interfaces específicas en lugar de una única interfaz inflada.

**Rompiendo ISP:**

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

Esta implementación fuerza a `OldPrinter` a implementar métodos que no soporta, violando ISP.

**Corrigiendo ISP:**

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
Dividiendo las funcionalidades en interfaces separadas, permitimos que los dispositivos implementen solo lo que necesitan.

---

#### 5. Principio de Inversión de Dependencias (DIP)
**Los módulos de alto nivel no deben depender de módulos de bajo nivel. Ambos deben depender de abstracciones.**

Este principio reduce el acoplamiento entre los módulos de alto y bajo nivel al introducir abstracciones.

**Rompiendo DIP:**

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

Aquí, `NotificationSender` está fuertemente acoplado a `EmailService`, lo que dificulta cambiar a un servicio de notificación diferente.

**Corrigiendo DIP:**

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
Aquí, `NotificationSender` depende de la abstracción `NotificationService`, haciéndolo flexible para trabajar con cualquier tipo de notificación.

---

### Conclusión
Los principios SOLID forman la base para construir software robusto y escalable. Kotlin, con su sintaxis expresiva y características modernas, permite a los desarrolladores implementar estos principios de manera elegante. Al adherirse a estos principios, puedes crear código que sea más fácil de mantener, extender y adaptar a los cambios en los requisitos.
