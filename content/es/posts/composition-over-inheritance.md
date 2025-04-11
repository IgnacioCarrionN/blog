---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Composición sobre Herencia: Una Perspectiva de Kotlin"
date: 2025-04-11T08:00:00+01:00
description: "Entendiendo las ventajas de la composición sobre la herencia con ejemplos prácticos en Kotlin."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/composition-inheritance.png
draft: false
tags: 
- kotlin
- arquitectura
- patrones de diseño
- programación orientada a objetos
---

### Composición sobre Herencia: Una Perspectiva de Kotlin

En la programación orientada a objetos, existen dos formas principales de reutilizar código y establecer relaciones entre clases: herencia y composición. Aunque ambos enfoques tienen su lugar, el principio de "composición sobre herencia" ha ganado una tracción significativa en el diseño de software moderno. Esta entrada de blog explora ambos enfoques, sus compensaciones, y por qué la composición es a menudo la opción preferida, con ejemplos en Kotlin.

---

#### Entendiendo la Herencia

La herencia es un mecanismo donde una clase (subclase) puede heredar propiedades y comportamientos de otra clase (superclase). Establece una relación "es-un" entre clases.

```kotlin
// Clase base
open class Animal {
    open fun makeSound() {
        println("Some generic animal sound")
    }

    fun eat() {
        println("Eating...")
    }
}

// Clase derivada
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
    dog.makeSound() // Salida: Woof!
    dog.eat()       // Salida: Eating...
    dog.fetch()     // Salida: Fetching...
}
```

En este ejemplo, `Dog` hereda de `Animal` y puede usar sus métodos mientras también añade su propio comportamiento.

**Ventajas de la Herencia:**
1. Reutilización de código - Las subclases heredan automáticamente métodos y propiedades
2. Sobrescritura de métodos - Permite la personalización del comportamiento heredado
3. Polimorfismo - Permite tratar objetos de diferentes subclases como objetos de la superclase

**Desventajas de la Herencia:**
1. Acoplamiento fuerte - Los cambios en la superclase pueden romper las subclases
2. Problema de la clase base frágil - Las modificaciones a la clase base pueden tener efectos inesperados
3. Inflexibilidad - La jerarquía de herencia se fija en tiempo de compilación
4. Limitada a herencia simple en muchos lenguajes (incluyendo Kotlin)

---

#### Entendiendo la Composición

La composición es un principio de diseño donde las clases logran comportamiento polimórfico y reutilización de código conteniendo instancias de otras clases en lugar de heredar de ellas. Establece una relación "tiene-un".

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

    dog.makeSound() // Salida: Woof!
    cat.makeSound() // Salida: Meow!
}
```

En este ejemplo, en lugar de heredar comportamiento, las clases `Dog` y `Cat` componen su comportamiento conteniendo instancias de `SoundBehavior` y `EatingBehavior`.

**Ventajas de la Composición:**
1. Flexibilidad - Los comportamientos pueden cambiarse en tiempo de ejecución
2. Desacoplamiento - Las clases son menos dependientes entre sí
3. No hay problema de clase base frágil - Los cambios en un componente no afectan a otros
4. Múltiples comportamientos - Puede incorporar múltiples comportamientos sin herencia múltiple

---

#### El Problema del Diamante y Por Qué Importa

Uno de los problemas clásicos con la herencia es el "problema del diamante", que ocurre en escenarios de herencia múltiple:

```
    A
   / \
  B   C
   \ /
    D
```

Si tanto B como C sobrescriben un método de A, ¿qué versión debería heredar D?

Aunque Kotlin no soporta herencia múltiple de clases, sí soporta implementación múltiple de interfaces, lo que puede llevar a problemas similares:

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

// Esto no compilará sin sobrescribir explícitamente doSomething
class D : B, C {
    override fun doSomething() {
        super<B>.doSomething() // Debemos elegir cuál llamar
    }
}
```

La composición evita este problema por completo haciendo las relaciones explícitas:

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

#### Ejemplo del Mundo Real: Componentes de UI

Veamos un ejemplo más práctico que involucra componentes de UI:

**Enfoque de Herencia:**

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

**Enfoque de Composición:**

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

// Uso
fun main() {
    val standardButton = UIComponent(StandardRenderer(), StandardClickHandler())
    val animatedButton = UIComponent(AnimatedRenderer(), StandardClickHandler())

    standardButton.render()    // Salida: Standard rendering
    animatedButton.render()    // Salida: Animated rendering
}
```

Con la composición, podemos mezclar y combinar comportamientos sin crear una jerarquía de herencia compleja. Incluso podemos cambiar comportamientos en tiempo de ejecución:

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

#### Cuándo Usar Herencia

A pesar de las ventajas de la composición, la herencia todavía tiene su lugar:

1. Cuando hay una clara relación "es-un" que es poco probable que cambie
2. Cuando quieres aprovechar el polimorfismo de manera directa
3. Cuando la clase base es estable y es poco probable que cambie frecuentemente
4. Para el diseño de frameworks donde los puntos de extensión están bien definidos

Por ejemplo, en la biblioteca estándar de Kotlin, `ArrayList` hereda de `AbstractList`, lo que tiene sentido porque un ArrayList es fundamentalmente una lista y esta relación no cambiará.

---

#### Mejores Prácticas

1. **Favorece la composición sobre la herencia** como regla general
2. **Usa herencia cuando hay una verdadera relación "es-un"** que sea estable
3. **Diseña para la composición** creando interfaces y clases pequeñas y enfocadas
4. **Considera la delegación** como un punto intermedio (Kotlin tiene soporte incorporado con la palabra clave `by`)
5. **Evita jerarquías de herencia profundas** ya que se vuelven difíciles de entender y mantener
6. **Programa hacia interfaces, no implementaciones** para facilitar la composición

La característica de delegación de Kotlin proporciona una forma conveniente de implementar la composición:

```kotlin
interface SoundMaker {
    fun makeSound()
}

class Barker : SoundMaker {
    override fun makeSound() = println("Woof!")
}

// Usando delegación con la palabra clave 'by'
class Dog(soundMaker: SoundMaker) : SoundMaker by soundMaker {
    // Métodos adicionales específicos de perro
    fun fetch() = println("Fetching...")
}

fun main() {
    val dog = Dog(Barker())
    dog.makeSound() // Salida: Woof!
    dog.fetch()     // Salida: Fetching...
}
```

---

### Conclusión

Aunque la herencia es una característica poderosa de la programación orientada a objetos, la composición a menudo proporciona un enfoque más flexible y mantenible para la reutilización de código y las relaciones entre clases. Al entender las compensaciones entre estos enfoques, puedes tomar mejores decisiones de diseño en tus proyectos de Kotlin.

Recuerda que un buen diseño no se trata de seguir reglas dogmáticamente, sino de elegir la herramienta adecuada para el trabajo. En muchos casos, esa herramienta será la composición, pero todavía hay casos de uso válidos para la herencia. La clave es entender las implicaciones de tu elección y diseñar tu código para que sea lo más flexible y mantenible posible.