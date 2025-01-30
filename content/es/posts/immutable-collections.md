---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Explorando la Librería de Colecciones Inmutables de Kotlin"
date: 2025-01-30T08:00:00+01:00
description: "Explorando la Librería de Colecciones Inmutables de Kotlin, usala en Compose para mejorar el rendimiento."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/readonly-list.png
draft: false
tags:
- kotlin
- collections
- compose
---

# Explorando la Librería de Colecciones Inmutables de Kotlin

Las colecciones estándar de Kotlin (`List`, `Set`, `Map`) son mutables por defecto, lo que puede provocar modificaciones no deseadas. Para garantizar la inmutabilidad a nivel de API, JetBrains introdujo la **librería de Colecciones Inmutables de Kotlin**. Esta librería proporciona un conjunto de tipos de colección verdaderamente inmutables que evitan modificaciones accidentales y mejoran la seguridad en entornos concurrentes o de múltiples hilos.

---

## ¿Por qué usar colecciones inmutables?

Aunque Kotlin ya ofrece `listOf()`, `setOf()` y `mapOf()` para colecciones de solo lectura, estas **no son verdaderamente inmutables**. La colección subyacente aún puede modificarse si se referencia en otro lugar. Ejemplo:

```kotlin
val list = mutableListOf("A", "B", "C")
val readOnlyList: List<String> = list  
list.add("D")  // Modifies the original list  
println(readOnlyList) // Output: [A, B, C, D]  

(readOnlyList as MutableList).add("E")
println(readOnlyList) // Output: [A, B, C, D, E]
```

Para resolver esto, la **librería de Colecciones Inmutables** proporciona colecciones que garantizan la inmutabilidad en tiempo de ejecución.

---

## Características clave

1. **Verdaderamente inmutables** – Una vez creadas, no pueden modificarse.
2. **Seguras para múltiples hilos** – Evitan modificaciones no intencionadas en entornos concurrentes.
3. **Optimizadas para el rendimiento** – Utilizan **compartición estructural** para evitar copias innecesarias.

---

## ¿Cómo usar las Colecciones Inmutables de Kotlin?

### 1. Agregar la dependencia

Primero, incluye la dependencia de **Colecciones Inmutables** en tu `build.gradle.kts`:

```kotlin
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-collections-immutable:0.3.5")
}
```

### 2. Crear colecciones inmutables

La librería proporciona `persistentListOf()`, `persistentSetOf()` y `persistentMapOf()` para crear colecciones inmutables:

```kotlin
import kotlinx.collections.immutable.*

val immutableList = persistentListOf("A", "B", "C")
val immutableSet = persistentSetOf(1, 2, 3)
val immutableMap = persistentMapOf("key1" to 100, "key2" to 200)
```

### 3. Agregar y eliminar elementos

Dado que estas colecciones son inmutables, las operaciones de modificación devuelven **una nueva copia modificada** en lugar de cambiar la colección original:

```kotlin
val newList = immutableList.add("D") // Creates a new list
println(newList)  // Output: [A, B, C, D]

val newMap = immutableMap.put("key3", 300)
println(newMap)   // Output: {key1=100, key2=200, key3=300}
```

La `immutableList` y `immutableMap` originales permanecen sin cambios.

---

## Consideraciones de rendimiento

A diferencia de las colecciones inmutables estándar (que requieren copias completas para modificaciones), las **colecciones persistentes** utilizan **compartición estructural**. Esto significa que las modificaciones crean una nueva colección reutilizando las partes no modificadas de la original, mejorando el rendimiento y la eficiencia de la memoria.

Por ejemplo, agregar un elemento a una lista persistente no crea una copia completa, sino que reutiliza la mayor parte de la estructura existente:

```
Original:  [A, B, C]  
New List:  [A, B, C, D] (Only "D" is newly allocated)  
```

Esto hace que las colecciones inmutables sean eficientes incluso para grandes conjuntos de datos.

---

## Beneficios en Jetpack Compose

Las colecciones inmutables son especialmente útiles en **Jetpack Compose**, ya que optimizan **la gestión del estado y las recomposiciones**. A continuación, se explica por qué son importantes en las aplicaciones Compose:

### 1. Evitan recomposiciones innecesarias

- Compose rastrea los cambios de estado para decidir cuándo recomponer los elementos de la UI.
- Las listas, conjuntos o mapas mutables pueden provocar **recomposiciones innecesarias** incluso cuando los datos no han cambiado.
- Las colecciones inmutables garantizan que el estado permanezca **estable**, evitando recomposiciones redundantes.

**Ejemplo:**

```kotlin
@Composable
fun MyListScreen(items: List<String>) {
    LazyColumn {
        items(items) { item ->
            Text(text = item)
        }
    }
}
```

Si `items` es una **lista mutable**, incluso reasignando los mismos valores **se activa una recomposición**. Usar una **colección inmutable** como `PersistentList` garantiza que Compose reconozca cuando los datos no han cambiado:

```kotlin
val items = remember { persistentListOf("A", "B", "C") }
MyListScreen(items)
```

### 2. Estabilidad del estado para mejorar el rendimiento

- Compose optimiza el renderizado omitiendo recomposiciones cuando los objetos de estado son **estables**.
- Las colecciones inmutables usan **compartición estructural**, lo que significa que las modificaciones solo afectan la parte cambiada mientras reutilizan el resto.
- Esto mejora el rendimiento en **listas grandes o jerarquías de UI complejas**.

### 3. Comportamiento predecible de la UI

- Dado que las colecciones inmutables **no pueden modificarse** después de la creación, evitan mutaciones accidentales que podrían generar actualizaciones impredecibles en la UI.
- Esto es especialmente útil en **arquitecturas dirigidas por estado (MVI, Redux, etc.)**, asegurando que la UI se actualice solo cuando sea necesario.

### 4. Seguridad en múltiples hilos

- En aplicaciones Compose que usan **corrutinas (Flows, LiveData, etc.)**, las colecciones inmutables previenen condiciones de carrera cuando múltiples hilos actualizan el estado.
- Garantizan un flujo de datos seguro entre **ViewModels, repositorios y componentes de UI**.

---

## ¿Cuándo usar colecciones inmutables?

✅ **Programación funcional** – Fomenta la inmutabilidad para transformaciones de datos más seguras.\
✅ **Seguridad en múltiples hilos** – Evita modificaciones no intencionadas en entornos concurrentes.\
✅ **Prevención de errores** – Reduce efectos secundarios inesperados debido a mutaciones accidentales.\
✅ **Gestión del estado en Compose** – Ayuda a optimizar recomposiciones y mejora el rendimiento de la UI.

---

## Conclusión

La librería de Colecciones Inmutables de Kotlin proporciona colecciones **verdaderamente inmutables**, **eficientes** y **seguras**, lo que las convierte en una excelente opción para la programación funcional, aplicaciones concurrentes y desarrollo con Jetpack Compose. Al aprovechar **colecciones persistentes**, puedes escribir código Kotlin más seguro y predecible.

🚀 **¿Usarías colecciones inmutables en tus proyectos?**
