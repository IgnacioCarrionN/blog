---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Abstracciones de Costo Cero en Kotlin: Funciones Inline y Value Classes"
date: 2025-10-24T08:00:00+01:00
description: "Guía práctica sobre funciones inline y Value Classes en Kotlin: cómo funcionan, por qué importan y cuándo usarlas para código más seguro y rápido."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/inline.png
draft: false
tags: 
- kotlin
- funciones-inline
- value-classes
- rendimiento
- genericos
---

### Abstracciones de Costo Cero en Kotlin: Funciones Inline y Value Classes

Kotlin ofrece dos herramientas muy poderosas para escribir código más seguro y rápido con sobrecarga nula o casi nula en tiempo de ejecución: las funciones inline y las Value Classes. Usadas correctamente, ayudan a evitar asignaciones, mejorar la seguridad de tipos y mantener APIs expresivas.

En este artículo veremos qué son, cómo funcionan por dentro, casos de uso prácticos, compromisos y cuándo no usarlas.

---

#### TL;DR

- Las funciones inline eliminan la sobrecarga en el sitio de llamada para utilidades de orden superior y habilitan parámetros de tipo reified.
- Las Value Classes envuelven un único valor con seguridad de tipos fuerte y (a menudo) sin asignaciones en tiempo de ejecución.
- Usa inline para funciones pequeñas y frecuentes, especialmente si toman lambdas o si reified simplifica la API.
- Usa Value Classes para tipos del dominio (UserId, Email, Money) que evitan confusiones con primitivos.
- Cuidado con el boxing de Value Classes con genéricos y nulabilidad, y con sobreusar inline en funciones grandes.

---

#### Funciones inline: qué son y por qué

Una función inline le pide al compilador copiar su cuerpo directamente en cada sitio de llamada. Esto es útil para funciones de orden superior (que reciben lambdas) porque puede evitar la asignación de objetos función y el coste de invocación indirecta.

Ejemplo básico:

```kotlin
inline fun <T> measure(label: String, block: () -> T): T {
    val start = systemTimeMillis()
    try {
        return block()
    } finally {
        val elapsed = systemTimeMillis() - start
        log("$label took ${elapsed}ms")
    }
}

fun demo() {
    val result = measure("expensive") {
        computeHeavy()
    }
}
```

Modificadores clave:

- noinline: evita que un parámetro lambda específico sea inline.
- crossinline: prohíbe retornos no locales desde una lambda (útil cuando la re-invocas).

```kotlin
inline fun process(
    crossinline action: () -> Unit,
    noinline onError: (Throwable) -> Unit
) {
    try {
        action() // re-invoked later or in another context → crossinline needed
    } catch (t: Throwable) {
        onError(t) // kept as a normal function value → noinline
    }
}
```

Parámetros de tipo reified

Normalmente, los genéricos se borran en JVM. Las funciones inline permiten usar parámetros de tipo reified, accediendo al tipo real en tiempo de ejecución.

```kotlin
inline fun <reified T> castOrNull(any: Any?): T? =
    if (any is T) any else null

val s: String? = castOrNull("hello") // works thanks to reified T
```

Cuándo usar funciones inline

- Utilidades muy pequeñas llamadas con frecuencia en rutas críticas
- Funciones de orden superior que de otra forma asignarían lambdas
- APIs que se benefician de tipos reified (helpers de JSON, utilidades sin reflexión pesada)

Cuándo evitar

- Funciones grandes: el inlining duplica código en los sitios de llamada y aumenta el tamaño del binario
- Funciones poco usadas o que no son críticas de rendimiento
- APIs públicas donde el tamaño del binario e inlining entre módulos importan

---

#### Value Classes: qué son y por qué

Una Value Class envuelve un único valor para dar seguridad de tipos con sobrecarga mínima o nula. En objetivos soportados, el compilador puede representarla como su valor subyacente en tiempo de ejecución, evitando asignaciones extra.

```kotlin
@JvmInline
value class UserId(val value: String) {
    init { require(value.isNotBlank()) { "UserId cannot be blank" } }
}

@JvmInline
value class Cents(val value: Long) {
    operator fun plus(other: Cents) = Cents(this.value + other.value)
    operator fun minus(other: Cents) = Cents(this.value - other.value)
}

fun pay(userId: UserId, amount: Cents) { /* ... */ }
```

Beneficios

- Seguridad de tipos: evita mezclar parámetros con el mismo primitivo (p. ej., UserId vs. OrderId)
- Modelado de dominio: haz que los estados ilegales sean irrepresentables (p. ej., NonEmptyString)
- Rendimiento: a menudo sin asignación extra respecto al primitivo

Wrappers comunes del dominio

```kotlin
@JvmInline value class OrderId(val value: String)
@JvmInline value class Email(val value: String)
@JvmInline value class Percentage private constructor(val value: Int) {
    companion object { fun of(raw: Int) = Percentage(raw.coerceIn(0, 100)) }
}
```

Trampas y casos borde

- Puede haber boxing con genéricos, interfaces y cuando el tipo es nullable (UserId?).
- Cuidado con reflexión y serialización; puede que necesites adaptadores.
- No abuses de las Value Classes como data classes: solo tienen una propiedad.

---

#### Casos de uso prácticos

1) IDs más seguros en APIs

```kotlin
@JvmInline value class ProductId(val value: String)
@JvmInline value class UserId(val value: String)

fun linkUserToProduct(userId: UserId, productId: ProductId) { /* ... */ }
```

2) Dinero y unidades

```kotlin
@JvmInline value class Money(val cents: Long) {
    operator fun plus(other: Money) = Money(cents + other.cents)
}

@JvmInline value class Meters(val value: Double)
```

3) Helpers de serialización con reified inline

```kotlin
inline fun <reified T> parse(json: String): T {
    // Usually delegated to your JSON library using T::class
    // Placeholder implementation:
    error("Provide adapter for ${T::class}")
}
```

4) Utilidades funcionales sin asignación

```kotlin
inline fun <T> T.alsoIf(condition: Boolean, crossinline block: (T) -> Unit): T {
    if (condition) block(this)
    return this
}
```

---

#### Notas de rendimiento

- JVM: Las Value Classes suelen optimizarse; aparece boxing con genéricos, interfaces o cuando son nullables. Mide en tu contexto.
- Kotlin/Native: Representación eficiente; el inlining reduce sobrecarga de llamada.
- JS: La representación difiere; valida con benchmarks.
- El inlining incrementa el tamaño del binario; evita inlining de cuerpos grandes usados en muchos sitios.

---

#### Interoperabilidad

- JVM y Android: Usa @JvmInline para mejor interop; ojo con llamadas desde Java viendo el tipo subyacente.
- Multiplataforma: Value Classes y funciones inline están soportadas en common; revisa el boxing específico por plataforma.
- Librerías: Si expones Value Classes, documenta serialización e interop (cómo las trata tu librería JSON).

---

#### Consejos de pruebas

- Prueba invariantes de tus Value Classes (constructores, operadores).
- Benchmarks en rutas críticas al introducir inline; confirma ganancias con tu carga real.
- Para serialización, añade pruebas de ida y vuelta para Value Classes.

---

#### Cuándo no usar

- Value Classes que esconden estado complejo (más de una propiedad) → usa data classes.
- Funciones inline con cuerpos grandes o poco usadas.
- APIs públicas donde consumidores de otros lenguajes (Java, Swift) puedan confundirse por el wrapper.

---

#### Resumen

- Las funciones inline eliminan indireccionamiento y habilitan genéricos reified.
- Las Value Classes mejoran la seguridad de tipos con coste mínimo en tiempo de ejecución.
- Juntas, permiten APIs expresivas, seguras y eficientes en Kotlin.

---

