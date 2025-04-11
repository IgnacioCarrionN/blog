---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Composition Over Inheritance: A Kotlin Perspective"
date: 2025-04-11T08:00:00+01:00
description: "Understanding the advantages of composition over inheritance with practical Kotlin examples."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/composition-inheritance.png
draft: false
tags: 
- kotlin
- architecture
- design patterns
- object-oriented programming
---

### Composition Over Inheritance: A Kotlin Perspective

In object-oriented programming, there are two primary ways to reuse code and establish relationships between classes: inheritance and composition. While both approaches have their place, the principle of "composition over inheritance" has gained significant traction in modern software design. This blog post explores both approaches, their trade-offs, and why composition is often the preferred choice, with examples in Kotlin.

---

#### Understanding Inheritance

Inheritance is a mechanism where a class (subclass) can inherit properties and behaviors from another class (superclass). It establishes an "is-a" relationship between classes.

```kotlin
// Base class
open class Animal {
    open fun makeSound() {
        println("Some generic animal sound")
    }

    fun eat() {
        println("Eating...")
    }
}

// Derived class
class Dog : Animal() {
    override fun makeSound() {
        println("Woof!")
    }

    fun fetch() {
        println("Fetching...")
    }
}

fun main() {
    val dog = Dog()
    dog.makeSound() // Outputs: Woof!
    dog.eat()       // Outputs: Eating...
    dog.fetch()     // Outputs: Fetching...
}
```

In this example, `Dog` inherits from `Animal` and can use its methods while also adding its own behavior.

**Advantages of Inheritance:**
1. Code reuse - Subclasses automatically inherit methods and properties
2. Method overriding - Allows customization of inherited behavior
3. Polymorphism - Enables treating objects of different subclasses as objects of the superclass

**Disadvantages of Inheritance:**
1. Tight coupling - Changes in the superclass can break subclasses
2. Fragile base class problem - Modifications to the base class can have unexpected effects
3. Inflexibility - The inheritance hierarchy is fixed at compile time
4. Limited to single inheritance in many languages (including Kotlin)

---

#### Understanding Composition

Composition is a design principle where classes achieve polymorphic behavior and code reuse by containing instances of other classes rather than inheriting from them. It establishes a "has-a" relationship.

```kotlin
interface SoundBehavior {
    fun makeSound()
}

class BarkSound : SoundBehavior {
    override fun makeSound() {
        println("Woof!")
    }
}

class MeowSound : SoundBehavior {
    override fun makeSound() {
        println("Meow!")
    }
}

class EatingBehavior {
    fun eat() {
        println("Eating...")
    }
}

class Dog(
    private val soundBehavior: SoundBehavior,
    private val eatingBehavior: EatingBehavior
) {
    fun makeSound() {
        soundBehavior.makeSound()
    }

    fun eat() {
        eatingBehavior.eat()
    }

    fun fetch() {
        println("Fetching...")
    }
}

class Cat(
    private val soundBehavior: SoundBehavior,
    private val eatingBehavior: EatingBehavior
) {
    fun makeSound() {
        soundBehavior.makeSound()
    }

    fun eat() {
        eatingBehavior.eat()
    }

    fun purr() {
        println("Purring...")
    }
}

fun main() {
    val eatingBehavior = EatingBehavior()
    val dog = Dog(BarkSound(), eatingBehavior)
    val cat = Cat(MeowSound(), eatingBehavior)

    dog.makeSound() // Outputs: Woof!
    cat.makeSound() // Outputs: Meow!
}
```

In this example, instead of inheriting behavior, the `Dog` and `Cat` classes compose their behavior by containing instances of `SoundBehavior` and `EatingBehavior`.

**Advantages of Composition:**
1. Flexibility - Behaviors can be changed at runtime
2. Decoupling - Classes are less dependent on each other
3. No fragile base class problem - Changes to one component don't affect others
4. Multiple behaviors - Can incorporate multiple behaviors without multiple inheritance

---

#### The Diamond Problem and Why It Matters

One of the classic problems with inheritance is the "diamond problem," which occurs in multiple inheritance scenarios:

```
    A
   / \
  B   C
   \ /
    D
```

If both B and C override a method from A, which version should D inherit?

While Kotlin doesn't support multiple inheritance of classes, it does support multiple interface implementation, which can lead to similar issues:

```kotlin
interface A {
    fun doSomething() {
        println("A's implementation")
    }
}

interface B : A {
    override fun doSomething() {
        println("B's implementation")
    }
}

interface C : A {
    override fun doSomething() {
        println("C's implementation")
    }
}

// This won't compile without explicitly overriding doSomething
class D : B, C {
    override fun doSomething() {
        super<B>.doSomething() // We must choose which one to call
    }
}
```

Composition avoids this problem entirely by making the relationships explicit:

```kotlin
class ComponentA {
    fun doSomething() {
        println("A's implementation")
    }
}

class ComponentB {
    fun doSomething() {
        println("B's implementation")
    }
}

class ComponentC {
    fun doSomething() {
        println("C's implementation")
    }
}

class D(
    private val componentB: ComponentB,
    private val componentC: ComponentC
) {
    fun doSomethingB() {
        componentB.doSomething()
    }

    fun doSomethingC() {
        componentC.doSomething()
    }
}
```

---

#### Real-World Example: UI Components

Let's look at a more practical example involving UI components:

**Inheritance Approach:**

```kotlin
open class UIComponent {
    open fun render() {
        println("Rendering component")
    }

    open fun handleClick() {
        println("Component clicked")
    }
}

open class Button : UIComponent() {
    override fun render() {
        println("Rendering button")
    }

    override fun handleClick() {
        println("Button clicked")
    }
}

class AnimatedButton : Button() {
    override fun render() {
        println("Rendering animated button")
    }
}
```

**Composition Approach:**

```kotlin
interface Renderer {
    fun render()
}

interface ClickHandler {
    fun handleClick()
}

class StandardRenderer : Renderer {
    override fun render() {
        println("Standard rendering")
    }
}

class AnimatedRenderer : Renderer {
    override fun render() {
        println("Animated rendering")
    }
}

class StandardClickHandler : ClickHandler {
    override fun handleClick() {
        println("Standard click handling")
    }
}

class UIComponent(
    private val renderer: Renderer,
    private val clickHandler: ClickHandler
) {
    fun render() {
        renderer.render()
    }

    fun handleClick() {
        clickHandler.handleClick()
    }
}

// Usage
fun main() {
    val standardButton = UIComponent(StandardRenderer(), StandardClickHandler())
    val animatedButton = UIComponent(AnimatedRenderer(), StandardClickHandler())

    standardButton.render()    // Outputs: Standard rendering
    animatedButton.render()    // Outputs: Animated rendering
}
```

With composition, we can mix and match behaviors without creating a complex inheritance hierarchy. We can even change behaviors at runtime:

```kotlin
class DynamicButton(
    private var renderer: Renderer,
    private var clickHandler: ClickHandler
) {
    fun render() {
        renderer.render()
    }

    fun handleClick() {
        clickHandler.handleClick()
    }

    fun setRenderer(newRenderer: Renderer) {
        renderer = newRenderer
    }

    fun setClickHandler(newClickHandler: ClickHandler) {
        clickHandler = newClickHandler
    }
}
```

---

#### When to Use Inheritance

Despite the advantages of composition, inheritance still has its place:

1. When there's a clear "is-a" relationship that's unlikely to change
2. When you want to leverage polymorphism in a straightforward way
3. When the base class is stable and unlikely to change frequently
4. For framework design where extension points are well-defined

For example, in Kotlin's standard library, `ArrayList` inherits from `AbstractList`, which makes sense because an ArrayList is fundamentally a list and this relationship won't change.

---

#### Best Practices

1. **Favor composition over inheritance** as a general rule
2. **Use inheritance when there's a true "is-a" relationship** that's stable
3. **Design for composition** by creating small, focused interfaces and classes
4. **Consider delegation** as a middle ground (Kotlin has built-in support with the `by` keyword)
5. **Avoid deep inheritance hierarchies** as they become difficult to understand and maintain
6. **Program to interfaces, not implementations** to make composition easier

Kotlin's delegation feature provides a convenient way to implement composition:

```kotlin
interface SoundMaker {
    fun makeSound()
}

class Barker : SoundMaker {
    override fun makeSound() = println("Woof!")
}

// Using delegation with 'by' keyword
class Dog(soundMaker: SoundMaker) : SoundMaker by soundMaker {
    // Additional dog-specific methods
    fun fetch() = println("Fetching...")
}

fun main() {
    val dog = Dog(Barker())
    dog.makeSound() // Outputs: Woof!
    dog.fetch()     // Outputs: Fetching...
}
```

---

### Conclusion

While inheritance is a powerful feature of object-oriented programming, composition often provides a more flexible and maintainable approach to code reuse and class relationships. By understanding the trade-offs between these approaches, you can make better design decisions in your Kotlin projects.

Remember that good design isn't about dogmatically following rules but about choosing the right tool for the job. In many cases, that tool will be composition, but there are still valid use cases for inheritance. The key is to understand the implications of your choice and to design your code to be as flexible and maintainable as possible.
