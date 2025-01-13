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

### **Exploring More Design Patterns in Kotlin: Part 3**

#### Design Patterns Series

- [Part 1](https://carrion.dev/en/posts/design-patterns-1/)
- [Part 2](https://carrion.dev/en/posts/design-patterns-2/)
- [Part 3](https://carrion.dev/en/posts/design-patterns-3/)

---

In this third installment, we’ll cover **Memento**, **Command**, **Visitor**, **Chain of Responsibility**, and **Mediator** patterns. These patterns address construction, behavioral, and structural challenges, showcasing Kotlin's expressive syntax and modern features.

---

#### **1. Memento Pattern**

The **Memento Pattern** captures and restores an object's state without exposing its internal details.

##### **When to Use**
- To implement undo/redo functionality.

##### **Kotlin Implementation**

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
    history.push(editor.save())

    editor.content = "Second Version"
    history.push(editor.save())

    editor.content = "Third Version"
    println("Current Content: ${editor.content}")

    editor.restore(history.pop()!!)
    println("Restored Content: ${editor.content}")

    editor.restore(history.pop()!!)
    println("Restored Content: ${editor.content}")
}
```

##### **Why Kotlin?**
Kotlin’s concise syntax makes state capture and restoration easy to implement.

---

#### **2. Command Pattern**

The **Command Pattern** encapsulates a request as an object, allowing for parameterization and queuing.

##### **When to Use**
- To implement undoable operations or command queues.

##### **Kotlin Implementation**

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

##### **Why Kotlin?**
Kotlin’s functional approach can further simplify command execution.

---

#### **3. Visitor Pattern**

The **Visitor Pattern** separates an algorithm from the object structure it operates on by moving the algorithm into a visitor object.

##### **When to Use**
- When you need to perform operations across a set of objects with varying types.

##### **Kotlin Implementation**

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

##### **Why Kotlin?**
Kotlin’s `fun interface` and sealed classes streamline the visitor implementation.

---

#### **4. Chain of Responsibility Pattern**

The **Chain of Responsibility Pattern** passes a request along a chain of handlers until one processes it.

##### **When to Use**
- When multiple objects can handle a request, and the handler isn’t determined until runtime.

##### **Kotlin Implementation**

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

##### **Why Kotlin?**
Kotlin’s nullable types and concise delegation simplify chaining handlers.

---

#### **5. Mediator Pattern**

The **Mediator Pattern** centralizes complex communication between multiple objects by having them communicate through a mediator.

##### **When to Use**
- When objects interact in complex ways, leading to tangled dependencies.

##### **Kotlin Implementation**

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

##### **Why Kotlin?**
Kotlin’s first-class functions and collections simplify broadcasting and interaction.

---

### **Conclusion**

These patterns—**Memento**, **Command**, **Visitor**, **Chain of Responsibility**, and **Mediator**—demonstrate Kotlin's ability to enhance classic design patterns with modern features.

Which of these patterns do you find most interesting? Let me know! 🚀
