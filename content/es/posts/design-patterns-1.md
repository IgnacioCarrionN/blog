---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Patrones de diseño en Kotlin - Parte 1"
date: 2024-12-30T08:00:00+01:00
description: "Kotlin Patrones de diseño - Parte 1"
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

### **Explorando patrones de diseño en Kotlin - Parte 1**

#### Serie Patrones de diseño

- [Parte 2](https://carrion.dev/es/posts/design-patterns-2/)

---

Los patrones de diseño son soluciones probadas a problemas comunes en el diseño de software. Con la sintaxis y funcionalidades modernas de Kotlin, implementar estos patrones normalmente resulta más limpio y conciso. En este post, exploraremos los patrones de **Singleton**, **Factory Method**, **Builder**, **Adapter** and **Decorator**, profundizando en su propósito, casos de uso y implementaciones en Kotlin.

---

#### **1. Patrón Singleton**

El **Patrón Singleton** asegura que una clase tiene solo una instancia y provee un punto de acceso global a ella.

##### **Cuando utilizar**

- Al manejar recursos compartidos como conexiones a bases de datos.

##### **Implementación en Kotlin**

La palabra reservada de Kotlin `object` provee una forma rápida de crear un Singleton.

```kotlin
object DatabaseConnection {
    fun connect() {
        println("Connecting to database...")
    }
}
```

##### **Uso**

```kotlin
fun main() {
    DatabaseConnection.connect()
}
```

##### **Ventajas en Kotlin**

- Por defecto es Thread-safe.
- Requiere un código mínimo comparado con implementaciones tradicionales en otros lenguajes.

---

#### **2. Patrón Factory Method**

El **Patrón Factory Method** delega la creación de objectos a clases o funciones, lo que provee de flexibilidad a la hora de instanciar los objetos.

##### **Cuando utilizarlo**

- Cuando crear los objetos requiere de lógica o tiene complejidad.
- Para desacoplar la creación del objeto del código del cliente.

##### **Implementación en Kotlin**

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

##### **Uso**

```kotlin
fun main() {
    val shape = ShapeFactory.createShape("Circle")
    shape.draw()
}
```

---

#### **3. Patrón Builder**

El **Patrón Builder** es usado para construir objetos complejos paso a paso. Es especialmente útil cuando un objeto tiene muchos parámetros opcionales o configuraciones distintas.

##### **Cuando utilizar**

- Para evitar constructores con demasiados parámetros.
- Cuando el proceso de contrucción del objeto es complejo o incluye multiples pasos.

##### **Implementación en Kotlin**

En Kotlin el uso de `apply` o las capacidades de `DSL` simplifican el patrón Builder.

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

##### **Uso**

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

##### ** Por qué en Kotlin?**

Enlazar métodos con `apply` permite una sintaxis más concisa y expresiva cuando se construye objetos.

---

#### **4. Patrón Adapter**

El **Patrón Adapter** es usado para hacer de puente entre interfaces que no son compatibles traduciendo una interfaz a la otra.

##### **Cuando utilizar**

- Cuando se integra nuevo código con código antiguo o librerías externas.
- Cuando dos sistemas o componentes necesitan trabajar en conjunto pero tienen interfaces incompatibles.

##### **Implementación en Kotlin**

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

##### **Uso**

```kotlin
fun main() {
    val intProvider = RandomIntProvider()
    val stringProvider: NewProvider = OldToNewProviderAdapter(intProvider)

    println(stringProvider.provideString())
}
```

##### **Por qué en Kotlin?**

Los constructores primaries de Kotlin y la sintaxis concisa simplifican la implementación de clases de tipo wrapper.

---

#### **5. Patrón Decorator**

El **Patrón Decorator** añade dinámicamente comportamientos a los objetos sin alterar su estructura.

##### **Cuando usarlo**

- Para extender la funcionalidad de una clase en tiempo de ejecución.
- Cuando heredar llevaría a una jerarquía sobrecargada.

##### **Implementación en Kotlin**

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

##### **Uso**

```kotlin
fun main() {
    val coffee = SimpleCoffee()
    val coffeeWithMilk = MilkDecorator(coffee)

    println("${coffeeWithMilk.description()} costs \$${coffeeWithMilk.cost()}")
}
```

---

### **Conclusión**

Las funcionalidades modernas de Kotlin como `object`, `when` y `apply` hacen que implementar los patrones de diseño tradicionales sea más fácil y expresivo. Estos patrones no solo resuelven desafíos comunes de diseño si no que demuestran como Kotlin mejora su implementación.

Hay otros patrones de diseño que te gustaria que cubriera en futuros posts?
