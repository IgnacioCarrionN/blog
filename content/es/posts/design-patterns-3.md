---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Kotlin Design Patterns - Part 3"
date: 2025-01-13T08:00:00+01:00
description: "Kotlin Design Patterns - Part 3"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/memento-pattern.png
draft: false
tags: 
- kotlin
- design-patterns
- architecture
---

### **Explorando Más Patrones de Diseño en Kotlin: Parte 3**

- [Part 1](https://carrion.dev/es/posts/design-patterns-1/)
- [Part 2](https://carrion.dev/es/posts/design-patterns-2/)
- [Part 3](https://carrion.dev/es/posts/design-patterns-3/)

---

En esta tercera entrega, cubriremos los patrones **Memento**, **Command**, **Visitor**, **Chain of Responsibility** y **Mediator**. Estos patrones abordan desafíos de construcción, comportamiento y estructura, mostrando la sintaxis expresiva y las características modernas de Kotlin.

---

#### **1. Patrón Memento**

El **Patrón Memento** captura y restaura el estado de un objeto sin exponer sus detalles internos.

##### **Cuándo Usar**
- Para implementar funcionalidad de deshacer/rehacer.

##### **Implementación en Kotlin**

```kotlin
class Editor {
    var content: String = ""

    fun createMemento(): Memento = Memento(content)
    fun restore(memento: Memento) { content = memento.state }

    data class Memento(val state: String)
}

class History {
    private val mementos = mutableListOf<Editor.Memento>()

    fun save(memento: Editor.Memento) {
        mementos.add(memento)
    }

    fun pop(): Editor.Memento? {
        if (mementos.isNotEmpty()) {
            return mementos.removeAt(mementos.lastIndex)
        }
        return null
    }
}

fun main() {
    val editor = Editor()
    val history = History()

    editor.content = "First Version"
    history.save(editor.createMemento())

    editor.content = "Second Version"
    history.save(editor.createMemento())

    editor.content = "Third Version"
    println("Current Content: ${editor.content}")

    editor.restore(history.pop()!!)
    println("Restored Content: ${editor.content}")

    editor.restore(history.pop()!!)
    println("Restored Content: ${editor.content}")
}
```

##### **Por Qué Kotlin?**
La sintaxis concisa de Kotlin facilita la captura y restauración de estados.

---

#### **2. Patrón Command**

El **Patrón Command** encapsula una solicitud como un objeto, permitiendo la parametrización y el encolado.

##### **Cuándo Usar**
- Para implementar operaciones deshacibles o colas de comandos.

##### **Implementación en Kotlin**

```kotlin
interface Command {
    fun execute()
}

class Light {
    fun on() = println("Light is ON")
    fun off() = println("Light is OFF")
}

class LightOnCommand(private val light: Light) : Command {
    override fun execute() = light.on()
}

class LightOffCommand(private val light: Light) : Command {
    override fun execute() = light.off()
}

fun main() {
    val light = Light()
    val commands = listOf(LightOnCommand(light), LightOffCommand(light))

    commands.forEach { it.execute() }
}
```

##### **Por Qué Kotlin?**
El enfoque funcional de Kotlin puede simplificar aún más la ejecución de comandos.

---

#### **3. Patrón Visitor**

El **Patrón Visitor** separa un algoritmo de la estructura de objetos sobre la que opera, moviendo el algoritmo a un objeto visitante.

##### **Cuándo Usar**
- Cuando necesitas realizar operaciones en un conjunto de objetos con tipos variados.

##### **Implementación en Kotlin**

```kotlin
interface Shape {
    fun accept(visitor: ShapeVisitor)
}

class Circle(val radius: Double) : Shape {
    override fun accept(visitor: ShapeVisitor) {
        visitor.visit(this)
    }
}

class Rectangle(val width: Double, val height: Double) : Shape {
    override fun accept(visitor: ShapeVisitor) {
        visitor.visit(this)
    }
}

fun interface ShapeVisitor {
    fun visit(shape: Shape)
}

fun main() {
    val shapes: List<Shape> = listOf(Circle(5.0), Rectangle(4.0, 6.0))

    val visitor = ShapeVisitor { shape ->
        when (shape) {
            is Circle -> println("Circle with radius ${shape.radius}")
            is Rectangle -> println("Rectangle with width ${shape.width} and height ${shape.height}")
        }
    }

    shapes.forEach { it.accept(visitor) }
}
```

##### **Por Qué Kotlin?**
Las `fun interface` y las clases selladas de Kotlin simplifican la implementación del visitante.

---

#### **4. Patrón Chain of Responsibility**

El **Patrón Chain of Responsibility** pasa una solicitud a lo largo de una cadena de manejadores hasta que uno la procesa.

##### **Cuándo Usar**
- Cuando múltiples objetos pueden manejar una solicitud y el handler se determina en tiempo de ejecución.

##### **Implementación en Kotlin**

```kotlin
interface Handler {
    fun handle(request: String): Boolean
}

class AuthHandler(private val next: Handler?) : Handler {
    override fun handle(request: String): Boolean {
        println("AuthHandler processing...")
        return next?.handle(request) ?: true
    }
}

class LoggingHandler(private val next: Handler?) : Handler {
    override fun handle(request: String): Boolean {
        println("LoggingHandler processing...")
        return next?.handle(request) ?: true
    }
}

fun main() {
    val chain = AuthHandler(LoggingHandler(null))
    chain.handle("Request")
}
```

##### **Por Qué Kotlin?**
Los tipos nulos de Kotlin y su delegación concisa simplifican el encadenamiento de handlers.

---

#### **5. Patrón Mediator**

El **Patrón Mediator** centraliza la comunicación compleja entre múltiples objetos haciendo que se comuniquen a través de un mediador.

##### **Cuándo Usar**
- Cuando los objetos interactúan de manera compleja, lo que lleva a dependencias enredadas.

##### **Implementación en Kotlin**

```kotlin
class Mediator {
    private val colleagues = mutableListOf<Colleague>()

    fun addColleague(colleague: Colleague) {
        colleagues.add(colleague)
    }

    fun broadcast(sender: Colleague, message: String) {
        colleagues.filter { it != sender }
            .forEach { it.receive(message) }
    }
}

interface Colleague {
    fun send(message: String)
    fun receive(message: String)
}

class ConcreteColleague(private val mediator: Mediator) : Colleague {
    override fun send(message: String) {
        println("Sending message: $message")
        mediator.broadcast(this, message)
    }

    override fun receive(message: String) {
        println("Received message: $message")
    }
}

fun main() {
    val mediator = Mediator()
    val colleague1 = ConcreteColleague(mediator)
    val colleague2 = ConcreteColleague(mediator)

    mediator.addColleague(colleague1)
    mediator.addColleague(colleague2)

    colleague1.send("Hello from Colleague 1")
}
```

##### **Por Qué Kotlin?**
Las funciones de primera clase y las colecciones de Kotlin simplifican la difusión y la interacción.

---

### **Conclusión**

Estos patrones—**Memento**, **Command**, **Visitor**, **Chain of Responsibility** y **Mediator**—demuestran la capacidad de Kotlin para mejorar patrones de diseño clásicos con características modernas.

¿Cuál de estos patrones encuentras más interesante? ¡Házmelo saber! 🚀

