---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Características Avanzadas de Genéricos y Varianza en Kotlin: Una Guía Completa"
date: 2025-03-21T08:00:00+01:00
description: "Domina los tipos genéricos avanzados y los conceptos de varianza en Kotlin con ejemplos prácticos y aplicaciones del mundo real"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/advanced-generics.png
draft: false
tags:
- kotlin
- generics
- variance
- type-safety
---

# Características Avanzadas de Genéricos y Varianza en Kotlin: Una Guía Completa

Entender los genéricos avanzados y la varianza en Kotlin es crucial para escribir código reutilizable y seguro en cuanto a tipos. Este artículo explora estos conceptos en profundidad, proporcionando ejemplos prácticos y aplicaciones del mundo real.

## Entendiendo la Varianza

La varianza en Kotlin determina cómo se relacionan los tipos genéricos con diferentes argumentos de tipo. Entender la varianza es más fácil cuando pensamos en términos de productores y consumidores:

- **Productor**: Solo produce/proporciona valores de tipo T (salida)
- **Consumidor**: Solo consume/acepta valores de tipo T (entrada)

Esta relación productor/consumidor se mapea directamente a los dos tipos de varianza:

- **Covarianza** (`out`): Se usa para productores - solo produce valores
- **Contravarianza** (`in`): Se usa para consumidores - solo consume valores

Así es como funcionan los productores y consumidores con tipos:

```
Type Hierarchy:    Producer<T>             Consumer<T>
Any               ▲ can produce            ▼ can consume
  └── Number      │ more specific         │ more general
       └── Int    │ types                 │ types
```

Por ejemplo:
- Un `Producer<Int>` puede usarse como `Producer<Number>` porque cualquier Int que produzca también es un Number
- Un `Consumer<Number>` puede usarse como `Consumer<Int>` porque cualquier cosa que pueda manejar Numbers puede manejar Ints

Una forma sencilla de recordarlo:
- Si una clase solo produce/devuelve T, hazla covariante con `out T` (puede usar tipos más específicos)
- Si una clase solo consume/acepta T, hazla contravariante con `in T` (puede usar tipos más generales)
- Si una clase produce y consume T, debe permanecer invariante (el tipo debe coincidir exactamente)

### Covarianza con out

La covarianza permite usar un tipo más derivado (específico) en lugar de un tipo menos derivado (general). En Kotlin, usamos el modificador `out` para indicar covarianza.

```kotlin
// Simple producer implementation
class IntProducer(private val value: Int) : Producer<Int> {
    override fun get(): Int = value
}

// Type safety with covariance:
// 1. IntProducer.get() returns Int
// 2. Int is a subtype of Number
// 3. Therefore, it's safe to use Producer<Int> as Producer<Number>
val intProducer: Producer<Int> = IntProducer(42)
val numberProducer: Producer<Number> = intProducer  // Safe: Int is always a Number

// Without 'out' modifier, this wouldn't compile:
// class RegularBox<T>(val item: T)  // invariant
// val intBox: RegularBox<Int> = RegularBox(42)
// val numBox: RegularBox<Number> = intBox  // Error: Type mismatch

// We can use the numberProducer wherever a Number is expected
fun printNumber(producer: Producer<Number>) {
    println("Number: ${producer.get()}")  // Safe: we know we'll get a Number
}

printNumber(intProducer)  // Works because Producer is covariant (out)
```

### Contravarianza con in

La contravarianza es lo opuesto a la covarianza. Permite usar un tipo más general donde se espera un tipo más específico. En Kotlin, usamos el modificador `in` para la contravarianza.

```kotlin
// Simple consumer implementation
class NumberProcessor : Consumer<Number> {
    override fun set(item: Number) {
        println("Processing number: ${item.toDouble()}")
    }
}

// Type safety with contravariance:
// 1. NumberProcessor.set() accepts any Number
// 2. Int is a subtype of Number
// 3. Therefore, it's safe to use Consumer<Number> as Consumer<Int>
val numberConsumer: Consumer<Number> = NumberProcessor()
val intConsumer: Consumer<Int> = numberConsumer     // Safe: anything that can handle Number can handle Int

// Without 'in' modifier, this wouldn't compile:
// class Processor<T>(val process: (T) -> Unit)  // invariant
// val numProcessor: Processor<Number> = Processor { println(it) }
// val intProcessor: Processor<Int> = numProcessor  // Error: Type mismatch

// We can use the intConsumer with Int values
fun processInt(consumer: Consumer<Int>) {
    consumer.set(42)  // Safe: we know the consumer can handle any Number, including Int
}

processInt(numberConsumer)  // Works because Consumer is contravariant (in)
```

Las restricciones tienen sentido porque:
- Un productor de Ints puede producirlos de forma segura donde se necesitan Numbers (cada Int es un Number)
- Un consumidor de Numbers puede consumir Ints de forma segura (sabe cómo manejar cualquier Number)

## Varianza en el Sitio de Declaración vs Varianza en el Sitio de Uso

Kotlin admite dos formas de especificar la varianza: varianza en el sitio de declaración (usando `in` o `out` en la declaración de clase/interfaz) y varianza en el sitio de uso (usando proyecciones de tipo). Cada enfoque tiene sus propios casos de uso y beneficios.

### Varianza en el Sitio de Declaración

La varianza en el sitio de declaración se especifica en la declaración del parámetro de tipo de una clase o interfaz. Este enfoque es preferible cuando una clase solo puede usar el parámetro de tipo de una manera en toda su implementación.

```kotlin
// Declaration-site variance example
interface Producer<out T> {
    fun produce(): T              // Can only produce/return T
    // fun consume(item: T) {}    // Error: Can't consume T in an out position
}

interface Consumer<in T> {
    fun consume(item: T)          // Can only consume T
    // fun produce(): T {}        // Error: Can't produce T in an in position
}

// Usage is straightforward - variance is handled automatically
class StringProducer : Producer<String> {
    override fun produce(): String = "Hello"
}

val producer: Producer<Any> = StringProducer() // OK: String is more specific than Any
```

### Varianza en el Sitio de Uso

La varianza en el sitio de uso (también conocida como proyección de tipo) se especifica en el punto de uso. Esto es útil cuando un tipo puede usarse tanto como productor como consumidor, pero en un uso específico, deseas restringirlo a un rol.

```kotlin
// Class with invariant type parameter
class Box<T>(var value: T) {
    fun get(): T = value
    fun set(value: T) { this.value = value }
}

// Use-site variance examples
fun copyOut(from: Box<out Number>, to: MutableList<Number>) {
    // 'from' is projected to be covariant (producer)
    // Can only call methods that return Number
    to.add(from.get())  // OK
    // from.set(42)     // Error: Can't call set on projected type
}

fun copyIn(to: Box<in Number>, from: List<Int>) {
    // 'to' is projected to be contravariant (consumer)
    // Can only call methods that accept Number
    to.set(from.first())  // OK
    // val x: Number = to.get()  // Error: Return type is projected to Nothing
}
```

### Cuándo Usar Cada Enfoque

1. Usa Varianza en el Sitio de Declaración Cuando:
   - La clase solo puede usar el parámetro de tipo de una manera (solo producir o solo consumir)
   - Quieres forzar el patrón de uso en todos los usos de la clase
   - El diseño de la API es claro sobre sus requisitos de varianza

```kotlin
// Good candidate for declaration-site variance
interface EventStream<out T> {
    fun next(): T
    fun peek(): T
    // Natural producer - only returns T
}
```

2. Usa Varianza en el Sitio de Uso Cuando:
   - La clase necesita tanto producir como consumir el tipo en general
   - Quieres restringir la varianza en puntos de uso específicos
   - Necesitas flexibilidad en cómo se usa el tipo en diferentes contextos

```kotlin
// Good candidate for use-site variance
class Stack<T> {
    private val items = mutableListOf<T>()

    // General implementation can both produce and consume T
    fun push(item: T) {
        items.add(item)
    }

    fun pop(): T {
        if (items.isEmpty()) throw NoSuchElementException("Stack is empty")
        return items.removeAt(items.lastIndex)
    }

    fun isEmpty(): Boolean = items.isEmpty()

    // But specific usages might want to restrict it:
    fun copyTo(other: Stack<in T>) {
        items.forEach { other.push(it) }
    }

    fun copyFrom(other: Stack<out T>) {
        // More efficient implementation using a temporary list
        val tempList = mutableListOf<T>()
        while (!other.isEmpty()) {
            tempList.add(other.pop())
        }
        tempList.asReversed().forEach { push(it) }
    }
}

// Usage examples demonstrating variance
fun main() {
    // Create stacks of different types
    val numberStack = Stack<Number>()
    val intStack = Stack<Int>()
    val doubleStack = Stack<Double>()

    // Fill stacks with values
    intStack.push(1)
    intStack.push(2)
    doubleStack.push(3.14)

    // Demonstrate contravariant usage with copyTo
    // Can copy from more specific type (Int) to more general type (Number)
    intStack.copyTo(numberStack)      // OK: Int is more specific than Number
    doubleStack.copyTo(numberStack)   // OK: Double is more specific than Number

    // Demonstrate covariant usage with copyFrom
    val intStack2 = Stack<Int>()
    val intStack3 = Stack<Int>()
    intStack2.push(42)
    intStack2.push(43)

    // Can copy from same type
    intStack3.copyFrom(intStack2)     // OK: same type

    // Can copy from more specific type to more general type
    val anyStack = Stack<Any>()
    anyStack.copyFrom(intStack)       // OK: Int is more specific than Any
    anyStack.copyFrom(doubleStack)    // OK: Double is more specific than Any

    // This demonstrates how use-site variance gives us flexibility:
    // - Stack<T> itself is invariant (can both read and write T)
    // - copyTo uses contravariance (in) to allow writing to more general types
    // - copyFrom uses covariance (out) to allow reading from more general types
}
```

### Proyecciones de Tipo

Las proyecciones de tipo son una forma de varianza en el sitio de uso que proporciona flexibilidad adicional al trabajar con tipos genéricos. Aquí hay una mirada más profunda a cómo funcionan:

```kotlin
fun copyElements(source: Array<out Number>, destination: Array<Number>) {
    for (i in source.indices) {
        destination[i] = source[i]
    }
}

val ints = arrayOf(1, 2, 3)
val numbers = Array<Number>(3) { 0.0 }
copyElements(ints, numbers) // Works thanks to out projection
```

### Restricciones Genéricas

Kotlin permite especificar límites superiores para los parámetros de tipo, restringiendo qué tipos se pueden usar.

```kotlin
interface Drawable {
    fun draw()
}

class Canvas<T : Drawable> {
    fun drawAll(elements: List<T>) {
        elements.forEach { it.draw() }
    }
}

class Circle : Drawable {
    override fun draw() = println("Drawing Circle")
}

val canvas = Canvas<Circle>()
canvas.drawAll(listOf(Circle(), Circle()))
```

### Restricciones Múltiples

Puedes especificar múltiples restricciones usando la cláusula where:

```kotlin
fun <T> copyWhenBothValid(
    source: T,
    destination: T
) where T : Drawable,
        T : Comparable<T> {
    if (source > destination) {
        destination.draw()
    }
}
```

## Mejores Prácticas y Directrices

1. Usa `out` cuando tu clase solo produce valores de tipo T
2. Usa `in` cuando tu clase solo consume valores de tipo T
3. Usa invarianza cuando tu clase produce y consume valores de tipo T
4. Prefiere la varianza en el sitio de declaración (`out`/`in` en la clase) sobre la varianza en el sitio de uso cuando sea posible
5. Usa proyecciones de estrella con moderación y solo cuando los argumentos de tipo son verdaderamente irrelevantes

## Errores Comunes y Soluciones

### Evitando Problemas de Borrado de Tipos

```kotlin
// Type erasure example
class TypeChecker {
    // Wrong way - won't compile due to type erasure
    fun isStringList(list: List<*>): Boolean {
        // return list is List<String>  // This won't compile
        return list.all { it is String }  // This is the correct way
    }

    // Correct way using reified type parameters
    inline fun <reified T> isListOf(list: List<*>): Boolean =
        list.all { it is T }
}
```

### Manejando Tipos Genéricos Nulables

```kotlin
// Explicitly handle nullable generic types
class Box<T : Any>(private var value: T?) {
    fun set(newValue: T) {
        value = newValue
    }

    fun get(): T? = value
}
```

## Conclusión

Los genéricos avanzados y la varianza en Kotlin proporcionan herramientas poderosas para construir abstracciones seguras y reutilizables. Al entender estos conceptos y aplicarlos apropiadamente, puedes escribir código más robusto y mantenible. Recuerda:

- Usar modificadores de varianza (`out`/`in`) cuando sea apropiado
- Aplicar restricciones genéricas para garantizar la seguridad de tipos
- Considerar tanto la varianza en el sitio de declaración como en el sitio de uso
- Ser consciente del borrado de tipos y la nulabilidad

El uso adecuado de estas características conduce a un código más elegante y seguro, reduciendo la probabilidad de errores en tiempo de ejecución y haciendo que tu base de código sea más mantenible.
