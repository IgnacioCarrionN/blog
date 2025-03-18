---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Patrones de Composición de Flows: Combinando Múltiples Flows de Manera Efectiva"
date: 2025-03-18T08:00:00+01:00
description: "Aprende a combinar múltiples Kotlin Flows de manera efectiva utilizando varios patrones de composición y mejores prácticas para construir pipelines de flows complejos"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/flow-composition.png
draft: false
tags:
- kotlin
- coroutines
- flows
- patterns
---

# Patrones de Composición de Flows: Combinando Múltiples Flows de Manera Efectiva

Cuando trabajamos con Kotlin Flows en aplicaciones del mundo real, a menudo necesitamos combinar múltiples flujos de datos para crear flujos de trabajo más complejos. Este artículo explora varios patrones de composición de Flows y las mejores prácticas para combinar múltiples Flows de manera efectiva.

## Entendiendo la Composición de Flows

La composición de Flows es el proceso de combinar múltiples Flows para crear un nuevo Flow que representa un flujo de datos más complejo. Kotlin proporciona varios operadores para la composición de Flows, cada uno sirviendo diferentes casos de uso.

### Operadores Básicos de Composición de Flows

Comencemos con los operadores fundamentales de composición de Flows:

```kotlin
// Sample data sources
val priceUpdates = flow { 
    emit(10.0)
    delay(100)
    emit(11.0)
}

val quantityUpdates = flow {
    emit(5)
    delay(200)
    emit(6)
}

// Combining latest values from both flows
val totalValueFlow = combine(priceUpdates, quantityUpdates) { price: Double, quantity: Int ->
    price * quantity
}
```

Output:
```
50.0  // 10.0 * 5
66.0  // 11.0 * 6
```

### Zip vs Combine

Los operadores `zip` y `combine` sirven para diferentes propósitos y tienen comportamientos distintos cuando trabajan con múltiples flows:

#### Operador Zip
- Empareja valores estrictamente uno a uno de cada flow
- Espera a que todos los flows emitan antes de producir un resultado
- Si un flow emite más lento, crea back-pressure (contrapresión)
- Útil cuando necesitas emparejar valores correspondientes de diferentes flows
- Si un flow se completa, el flow resultante también se completa

#### Operador Combine
- Utiliza el último valor de cada flow para producir resultados
- Emite cada vez que cualquier flow produce un nuevo valor
- Sin back-pressure - usa el valor más reciente de otros flows
- Útil para actualizaciones en tiempo real donde necesitas el último estado
- Continúa hasta que todos los flows se completen

Aquí hay un ejemplo práctico que muestra la diferencia:

```kotlin
// zip: Pairs corresponding values (one-to-one)
val zippedFlow = priceUpdates.zip(quantityUpdates) { price: Double, quantity: Int ->
    "Price: $price, Quantity: $quantity"
}

// combine: Emits when either flow emits (using latest values)
val combinedFlow = combine(priceUpdates, quantityUpdates) { price: Double, quantity: Int ->
    "Latest Price: $price, Latest Quantity: $quantity"
}
```

Veamos cómo se comportan con diferentes tiempos:

```kotlin
val prices = flow {
    emit(10.0)  // t=0ms
    delay(100)
    emit(11.0)  // t=100ms
    delay(100)
    emit(12.0)  // t=200ms
}

val quantities = flow {
    emit(5)     // t=0ms
    delay(150)
    emit(6)     // t=150ms
}
```

Output for zippedFlow (empareja valores en orden):
```
"Price: 10.0, Quantity: 5"    // Primer par
"Price: 11.0, Quantity: 6"    // Segundo par
// 12.0 nunca se emite porque quantities no tiene más valores
```

Output for combinedFlow (reacciona a cada cambio):
```
"Latest Price: 10.0, Latest Quantity: 5"    // Valores iniciales
"Latest Price: 11.0, Latest Quantity: 5"    // Precio actualizado en t=100ms
"Latest Price: 11.0, Latest Quantity: 6"    // Cantidad actualizada en t=150ms
"Latest Price: 12.0, Latest Quantity: 6"    // Precio actualizado en t=200ms
```

Este ejemplo muestra cómo:
- `zip` empareja valores en secuencia y requiere que ambos flows emitan
- `combine` reacciona a cambios en cualquier flow y usa los últimos valores disponibles
- `zip` puede omitir valores si los flows emiten a diferentes velocidades
- `combine` asegura que siempre trabajas con los datos más recientes

## Patrones de Composición Avanzados

### Fusionando Múltiples Flows

El operador `merge` combina múltiples flows en un único flow, preservando el tiempo relativo de emisión de cada fuente. A diferencia de `zip` o `combine`, `merge` simplemente reenvía los valores según llegan, sin intentar emparejarlos o combinarlos.

#### Características Principales de Merge
- Emite valores tan pronto como llegan de cualquier flow fuente
- Mantiene el orden de emisiones dentro de cada flow fuente
- No espera ni combina valores de diferentes flows
- Se completa solo cuando todos los flows fuente se completan
- Útil para manejar eventos independientes de múltiples fuentes

Aquí vemos cómo funciona merge con múltiples fuentes de eventos:

```kotlin
val userActions = merge(
    buttonClicks,
    menuSelections,
    gestureEvents
)

// Alternative using Flow builder
val mergedFlow = flow {
    coroutineScope {
        launch { buttonClicks.collect { emit(it) } }
        launch { menuSelections.collect { emit(it) } }
        launch { gestureEvents.collect { emit(it) } }
    }
}
```

Veamos cómo merge maneja eventos con diferentes tiempos:

```kotlin
val clickEvents = flow {
    emit("Click 1")  // t=0ms
    delay(100)
    emit("Click 2")  // t=100ms
}

val keyEvents = flow {
    delay(50)
    emit("Key A")    // t=50ms
    delay(100)
    emit("Key B")    // t=150ms
}

val gestureEvents = flow {
    delay(75)
    emit("Swipe")    // t=75ms
}

val allEvents = merge(clickEvents, keyEvents, gestureEvents)
```

Output (eventos en orden de llegada):
```
"Click 1"    // t=0ms   (desde clickEvents)
"Key A"      // t=50ms  (desde keyEvents)
"Swipe"      // t=75ms  (desde gestureEvents)
"Click 2"    // t=100ms (desde clickEvents)
"Key B"      // t=150ms (desde keyEvents)
```

Casos de Uso Comunes para Merge:
1. **Manejo de Eventos**: Combinación de interacciones de usuario desde diferentes fuentes
2. **Actualizaciones Multi-fuente**: Monitoreo de cambios desde múltiples fuentes de datos independientes
3. **Procesamiento Paralelo**: Recolección de resultados de operaciones paralelas
4. **Monitoreo de Sistema**: Agregación de logs o métricas desde múltiples componentes

La implementación alternativa usando un Flow builder muestra cómo funciona merge internamente:

## Manejo de Errores en Flows Compuestos

Cuando trabajamos con flows compuestos, el manejo de errores se vuelve particularmente importante ya que los errores pueden propagarse a través de la cadena de flows y afectar múltiples fuentes de datos. Existen varias estrategias para manejar errores en flows compuestos:

### 1. Manejo de Errores Individual por Flow
Cada flow puede manejar sus propios errores antes de la composición:

```kotlin
val safeFlow1 = flow1.catch { error: Throwable ->
    emit(fallbackValue)
    // o
    emit(Result.failure<String>(error))
}

val safeFlow2 = flow2.catch { error: Throwable ->
    // Registrar error y emitir valor por defecto
    logger.error("Flow 2 falló", error)
    emit(defaultValue)
}
```

### 2. Transformación de Errores en Flows Compuestos
Transformar errores en resultados específicos del dominio:

```kotlin
sealed class DataResult<T> {
    data class Success<T>(val data: T) : DataResult<T>()
    data class Error<T>(val error: Throwable) : DataResult<T>()
}

fun <T> Flow<T>.asResult(): Flow<DataResult<T>> = map { value: T -> 
    DataResult.Success(value) 
}.catch { e: Throwable ->
    emit(DataResult.Error<T>(e))
}

// Uso en composición
val combinedFlow = combine(
    flow1.asResult(),
    flow2.asResult()
) { result1: DataResult<Data1>, result2: DataResult<Data2> ->
    when {
        result1 is DataResult.Error<Data1> -> result1 as DataResult<CombinedData>
        result2 is DataResult.Error<Data2> -> result2 as DataResult<CombinedData>
        else -> DataResult.Success(
            combineData(
                (result1 as DataResult.Success<Data1>).data,
                (result2 as DataResult.Success<Data2>).data
            )
        )
    }
}
```

### 3. Usando el Tipo Result para Manejo de Errores
Un patrón común utilizando el tipo Result de Kotlin:

```kotlin
fun combineDataSources(): Flow<Result<CombinedData>> =
    combine(
        source1.catch { emit(Result.failure(it)) },
        source2.catch { emit(Result.failure(it)) }
    ) { result1: Result<Data1>, result2: Result<Data2> ->
        when {
            result1.isFailure -> result1 as Result<CombinedData>
            result2.isFailure -> result2 as Result<CombinedData>
            else -> Result.success(
                CombinedData(
                    result1.getOrNull()!!,
                    result2.getOrNull()!!
                )
            )
        }
    }
```

### 4. Estrategias de Reintento
Implementar lógica de reintento para fallos transitorios:

```kotlin
fun <T> Flow<T>.retryWithBackoff(
    maxAttempts: Int = 3,
    initialDelay: Long = 100,
    maxDelay: Long = 1000,
    factor: Double = 2.0
): Flow<T> = retry { cause: Throwable, attempt: Long ->
    if (attempt < maxAttempts) {
        delay(
            (initialDelay * factor.pow(attempt.toDouble()))
                .toLong().coerceAtMost(maxDelay)
        )
        true
    } else false
}

// Uso en flows compuestos
val resilientFlow = combine(
    flow1.retryWithBackoff(),
    flow2.retryWithBackoff()
) { value1: Data1, value2: Data2 ->
    // Procesar valores
    CombinedData(value1, value2)
}
```

### Mejores Prácticas para el Manejo de Errores en Flows Compuestos:

1. **Manejar Errores Cerca de la Fuente**:
   - Capturar errores en flows individuales antes de la composición
   - Transformar errores en resultados específicos del dominio
   - Proporcionar valores de respaldo cuando sea apropiado

2. **Estrategias de Recuperación de Errores**:
   - Implementar lógica de reintento para fallos transitorios
   - Usar estrategias de backoff para evitar sobrecargar los sistemas
   - Considerar proporcionar valores por defecto o en caché

3. **Propagación de Errores**:
   - Decidir si propagar o manejar errores localmente
   - Usar tipos de error estructurados (sealed classes, tipo Result)
   - Mantener el contexto del error a través de la cadena de flows

4. **Monitoreo y Depuración**:
   - Registrar errores con el contexto apropiado
   - Rastrear tasas y patrones de errores
   - Implementar un reporte de errores adecuado

## Consideraciones de Rendimiento

Cuando combinamos Flows, considera estas técnicas de optimización de rendimiento:

1. **Gestión de Buffer**:
```kotlin
val optimizedFlow = flow1
    .buffer(Channel.BUFFERED)
    .combine(flow2.buffer(Channel.BUFFERED)) { value1: T1, value2: T2 ->
        // Process values
    }
```

2. **Conflación para Últimos Valores**:
```kotlin
val conflatedFlow = flow1
    .conflate()
    .combine(flow2.conflate()) { value1: T1, value2: T2 ->
        // Process only latest values
    }
```

## Conclusión

La composición de Flows es una característica poderosa que te permite construir flujos reactivos complejos en Kotlin. Al comprender estos patrones y mejores prácticas, puedes combinar efectivamente múltiples Flows mientras mantienes un código limpio, mantenible y eficiente. Recuerda:

- Elegir el operador de composición adecuado para tu caso de uso
- Manejar errores apropiadamente en cada nivel
- Considerar las implicaciones de rendimiento
- Implementar un manejo adecuado de la cancelación

Estos patrones te ayudarán a construir aplicaciones robustas que puedan manejar flujos de datos complejos de manera efectiva.
