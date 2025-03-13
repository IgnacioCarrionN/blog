---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Entendiendo los Operadores de Flujo (Flow): Buffer, Conflate, Debounce y Sample"
date: 2024-01-09T08:00:00+01:00
description: "Análisis profundo de los operadores de flujo en Kotlin (Flow): buffer, conflate, debounce y sample. Aprende cuándo y cómo usar cada operador con ejemplos prácticos."
hideToc: false
enableToc: true
enableTocContent: false
image: /images/kotlin/flow-operators.png
draft: false
tags:
- kotlin
- flows
- coroutines
---

# Entendiendo los Operadores de Flujo (Flow): Buffer, Conflate, Debounce y Sample

Cuando trabajamos con flujos de Kotlin (Flows), especialmente en escenarios que involucran productores que emiten rápidamente y colectores lentos, es crucial entender cómo gestionar el flujo de datos de manera efectiva. Este post explora cuatro operadores esenciales de Flow que ayudan a manejar estos escenarios: `buffer`, `conflate`, `debounce` y `sample`.

## El Problema: Colectores Lentos

Antes de profundizar en los operadores, entendamos el problema que resuelven. Considera este escenario:

```kotlin
flow {
    for (i in 1..100) {
        emit(i)
        delay(100) // Emit every 100ms
    }
}.collect { value: Int ->
    delay(300) // Process for 300ms
    println("Processed $value")
}
```

En este caso, el colector es más lento que el productor, lo que puede llevar a problemas de contrapresión. Cada operador que discutiremos proporciona una estrategia diferente para manejar esta situación.

## Operador Buffer

El operador `buffer` crea un canal con capacidad específica para almacenar emisiones mientras el colector procesa los valores anteriores.

```kotlin
flow {
    for (i in 1..5) {
        emit(i)
        println("Emitting $i")
        delay(100)
    }
}.buffer(2) // Buffer capacity of 2
 .collect { value: Int ->
    println("Collecting $value")
    delay(300) // Slow collector
}

// Output:
// Emitting 1
// Emitting 2
// Emitting 3
// Collecting 1 (t=300ms)
// Collecting 2 (t=600ms)
// Emitting 4
// Emitting 5
// Collecting 3 (t=900ms)
// Collecting 4 (t=1200ms)
// Collecting 5 (t=1500ms)
```

### Cuándo Usar Buffer
- Cuando necesitas almacenar un número específico de emisiones
- Cuando necesitas procesar todos los valores pero quieres independizar las velocidades del productor y el colector
- Cuando es importante mantener el orden de procesamiento

## Operador Conflate

El operador `conflate` mantiene solo el último valor, descartando los valores intermedios si el colector no puede mantener el ritmo.

```kotlin
flow {
    for (i in 1..5) {
        emit(i)
        println("Emitting $i")
        delay(100)
    }
}.conflate()
 .collect { value: Int ->
    println("Collecting $value")
    delay(300) // Slow collector
}

// Output:
// Emitting 1
// Emitting 2
// Collecting 1 (t=300ms)
// Emitting 3
// Emitting 4
// Collecting 4 (t=600ms)
// Emitting 5
// Collecting 5 (t=900ms)
```

### Cuándo Usar Conflate
- Cuando solo interesa el valor más reciente
- En escenarios de interfaz de usuario donde no es necesario mostrar estados intermedios
- Cuando no es crítico procesar cada valor individual

## Operador Debounce

El operador `debounce` emite un valor solo después de que ha pasado un tiempo específico sin nuevas emisiones.

```kotlin
flow {
    emit(1)
    delay(90)
    emit(2) // Will be dropped
    delay(90)
    emit(3) // Will be dropped
    delay(150) // Longer than debounce timeout
    emit(4) // Will be emitted
}.debounce(100)
 .collect { value: Int ->
    println("Collecting $value")
}

// Output:
// Collecting 1 (t=100ms)
// Collecting 4 (t=430ms)
// (Values 2 and 3 are dropped because new values arrived before debounce timeout)
```

### Cuándo Usar Debounce
- Para implementar búsqueda mientras se escribe
- Para manejar eventos rápidos de interfaz de usuario
- Cuando necesitas esperar "períodos de calma" antes de procesar

## Operador Sample

El operador `sample` toma periódicamente el valor más reciente del flujo en intervalos específicos.

```kotlin
flow {
    var i = 0
    while (i < 10) {
        emit(i++)
        delay(50) // Emit every 50ms
    }
}.sample(100) // Sample every 100ms
 .collect { value: Int ->
    println("Sampled value: $value")
}

// Output:
// Sampled value: 1  (t=100ms)
// Sampled value: 3  (t=200ms)
// Sampled value: 5  (t=300ms)
// Sampled value: 7  (t=400ms)
// Sampled value: 9  (t=500ms)
// (Only captures the latest value every 100ms)
```

### Cuándo Usar Sample
- Cuando requieres actualizaciones regulares en intervalos fijos
- Para mostrar datos en tiempo real donde los valores intermedios no son relevantes
- Para limitar la tasa de procesamiento independientemente de la frecuencia de emisión

## Comparación y Mejores Prácticas

Aquí hay una comparación rápida de estos operadores:

| Operador  | Comportamiento | Caso de Uso |
|-----------|----------|-----------|
| buffer    | Almacena emisiones | Procesamiento completo manteniendo el orden |
| conflate  | Mantiene solo el último | Actualizaciones de interfaz, último valor |
| debounce  | Espera período de calma | Búsqueda en tiempo real, eventos rápidos |
| sample    | Muestreo periódico | Actualizaciones regulares, control de frecuencia |

## Conclusión

Comprender estos operadores de Flow es fundamental para construir aplicaciones reactivas eficientes:
- Utiliza `buffer` cuando necesites procesar todos los valores y controlar el uso de memoria
- Utiliza `conflate` cuando solo importe el último valor
- Utiliza `debounce` cuando manejes eventos rápidos que requieran "tiempo de asentamiento"
- Utiliza `sample` cuando necesites actualizaciones regulares en intervalos fijos

Selecciona el operador adecuado según tu caso de uso específico y los requisitos de completitud de datos, orden y frecuencia de procesamiento.

Recuerda que estos operadores se pueden combinar para crear flujos de procesamiento de datos más sofisticados, pero evita sobre-complicar tus flujos. Considera siempre el equilibrio entre la completitud de datos, el uso de memoria y la eficiencia de procesamiento.