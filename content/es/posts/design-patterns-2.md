---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Patrones de diseño en Kotlin - Parte 2"
date: 2025-01-06T08:00:00+01:00
description: "Kotlin Patrones de diseño - Parte 2"
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

### **Explorando patrones de diseño en Kotlin: Parte2**

Después de la gran acogida del primer artículo [Patrones de diseño en Kotlin](https://carrion.dev/es/posts/design-patterns-1/), volvemos con más! En esta segunda parte, revisaremos los patrones de **Prototype**, **Composite**, **Proxy**, **Observer**, y **Strategy**. Estos patrones resuelven una variedad de desafios de diseño y demuestran las capacidades expresivas de Kotlin.

---

#### **1. Patrón Prototype**

El **Patrón Prototype** es usado para crear nuevos objeto copiando una objeto existente, asegurando la creación eficaz de objetos.

##### **Cuando usarlo**

- Cuando crear una nueva instancia es complejo o costoso.
- Para evitar crear instancias de subclases de forma repetida.

##### **Implementación en Kotlin**

Usar las clases `data` de Kotlin y su función `copy` simplifica este patrón.

```kotlin
data class Document(var title: String, var content: String, var author: String)

fun main() {
    val original = Document("Design Patterns", "Content about patterns", "John Doe")
    val copy = original.copy(title = "Prototype Pattern")

    println("Original: $original")
    println("Copy: $copy")
}
```

##### **Por qué Kotlin?**

Las clases `data` de Kotlin soportan de forma nativa copiar los objetos con código mínimo, haciendo que aplicar el patrón Prototype sea muy sencillo.

---

#### **2. Patrón Composite**

El **Patrón Composite** es usado para tratar objetos individuales y grupos de forma uniforme.

##### **Cuando usarlo**

- Cuando tienes una estructura en árbol y quieres manipularlo de una forma consistente.

##### **Implementación en Kotlin**

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

#### **3. Patrón Proxy**

El **Patrón Proxy** sirve de puerta de entrada para controlar el acceso a otro objeto.

##### **Cuando utilizarlo**

- Para controlar el acceso a otro recurso.
- Para añadir funcionalidad sin modificar el objeto existente.

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

#### **4. Patrón Observer**

El **Patrón Observer** define una dependencia de uno-a-muchos, por lo que cuando un objeto cambia su estado, todos los que dependen de el son notificados.

##### **Cuando utilizarlo**

- Para sistemas dirigidos por eventos.
- Cuando múltiples componentes necesitan reaccionar a cambios de estado.

##### **Implementación en Kotlin**

```kotlin
import kotlin.properties.Delegates

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

##### **Por qué Kotlin?**

Usar `fun interface` simplifica la implementación de interfaces con un sólo método. De forma adicional, los `Delegates.observable` de Kotlin hace que observar cambios de estado sea más directo, facilitando la implementación del patrón Observer.

---

#### **5. Patrón Strategy**

El **Patrón Strategy** define una seria de algoritmos, encapsula cada uno de ellos, y luego los hace intercambiables.

##### **Cuando utilizar**

- Cuando necesitas vaerios algoritmos para una tarea en concreto.

##### **Implementación en Kotlin**

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

### **Conclusión**

Con Kotlin, los patrones de diseño como **Prototype**, **Composite**, **Proxy**, **Observer**, y **Strategy** se vuelven más intuitivos. Estos patrones no son solo herramientas, son los fundamentos para un código más claro y mantenible.
