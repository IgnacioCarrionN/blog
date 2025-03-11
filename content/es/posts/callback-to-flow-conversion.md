---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Convirtiendo Callbacks a Corrutinas y Flows en Kotlin"
date: 2025-03-11T08:00:00+01:00
description: "Aprende cómo transformar APIs basadas en callbacks a corrutinas y flows modernos de Kotlin usando suspendCoroutine y callbackFlow"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/suspend-coroutine.png
draft: false
tags:
- kotlin
- coroutines
- flows
- callbacks
---

# Convirtiendo Callbacks a Coroutines y Flows en Kotlin

Las APIs basadas en callbacks han sido un patrón común en la programación asíncrona durante muchos años. Sin embargo, con las corrutinas y flows de Kotlin, podemos transformar estos callbacks en código moderno y secuencial que es más fácil de leer y mantener. En este artículo, exploraremos cómo usar `suspendCoroutine` y `callbackFlow` para convertir APIs basadas en callbacks a corrutinas y flows.

## Entendiendo suspendCoroutine

La función `suspendCoroutine` es una herramienta poderosa que nos permite envolver APIs basadas en callbacks en funciones suspend. Esta transformación hace que el código asíncrono sea más secuencial y fácil de manejar.

### Uso Básico de suspendCoroutine

Aquí hay un ejemplo simple de cómo convertir una función basada en callbacks a una función suspend:

```kotlin
// Traditional callback-based API
interface LocationCallback {
    fun onLocationFound(location: Location)
    fun onError(error: Exception)
}

class LocationService {
    fun getCurrentLocation(callback: LocationCallback) {
        // Simulating async location fetch
        // Implementation details...
    }
}

// Converted to suspend function
suspend fun LocationService.getLocationSuspend(): Location {
    return suspendCoroutine<Location> { continuation: Continuation<Location> ->
        getCurrentLocation(object : LocationCallback {
            override fun onLocationFound(location: Location) {
                continuation.resume(location)
            }

            override fun onError(error: Exception) {
                continuation.resumeWithException(error)
            }
        })
    }
}

// Usage
suspend fun fetchLocation() {
    try {
        val location = locationService.getLocationSuspend()
        println("Ubicación recibida: $location")
    } catch (e: Exception) {
        println("Error al obtener la ubicación: ${e.message}")
    }
}
```

### Manejo de Cancelación

Cuando trabajamos con `suspendCoroutine`, es importante manejar la cancelación adecuadamente:

```kotlin
suspend fun LocationService.getLocationSuspendWithCancellation(): Location {
    return suspendCancellableCoroutine<Location> { continuation: CancellableContinuation<Location> ->
        val callback = object : LocationCallback {
            override fun onLocationFound(location: Location) {
                continuation.resume(location)
            }

            override fun onError(error: Exception) {
                continuation.resumeWithException(error)
            }
        }

        getCurrentLocation(callback)

        continuation.invokeOnCancellation {
            // Cleanup resources, remove callbacks, etc.
            removeLocationUpdates(callback)
        }
    }
}
```

## Convirtiendo a Flows con callbackFlow

Mientras que `suspendCoroutine` es excelente para operaciones únicas, `callbackFlow` es perfecto para manejar flujos de datos o eventos. Nos permite convertir APIs basadas en callbacks que emiten múltiples valores en Kotlin Flows.

### Ejemplo Básico de callbackFlow

Aquí te mostramos cómo convertir una API de actualizaciones de ubicación a un Flow:

```kotlin
interface LocationUpdatesCallback {
    fun onLocationUpdate(location: Location)
    fun onError(error: Exception)
}

class LocationService {
    fun startLocationUpdates(callback: LocationUpdatesCallback) {
        // Implementation details...
    }

    fun stopLocationUpdates(callback: LocationUpdatesCallback) {
        // Implementation details...
    }
}

fun LocationService.locationUpdatesFlow(): Flow<Location> = callbackFlow {
    val callback = object : LocationUpdatesCallback {
        override fun onLocationUpdate(location: Location) {
            trySend(location)
        }

        override fun onError(error: Exception) {
            close(error)
        }
    }

    startLocationUpdates(callback)

    // Clean up when the flow is cancelled
    awaitClose {
        stopLocationUpdates(callback)
    }
}

// Usage
suspend fun trackLocation() {
    locationService.locationUpdatesFlow()
        .catch { error: Throwable -> 
            println("Error en actualizaciones de ubicación: ${error.message}")
        }
        .collect { location: Location ->
            println("Nueva ubicación: $location")
        }
}
```

### Manejo de Contrapresión (Backpressure)

Cuando tratamos con actualizaciones frecuentes, es importante manejar la contrapresión adecuadamente:

```kotlin
fun SensorService.sensorUpdatesFlow(): Flow<SensorData> = callbackFlow {
    val callback = object : SensorCallback {
        override fun onSensorUpdate(data: SensorData) {
            // Use trySend instead of send to handle backpressure
            if (!trySend(data).isSuccess) {
                // Handle unsuccessful send
                println("Buffer full, dropping sensor update")
            }
        }
    }

    registerSensorCallback(callback)

    awaitClose {
        unregisterSensorCallback(callback)
    }
}.buffer(Channel.CONFLATED) // Keep only the latest value
```

## Mejores Prácticas

1. **Manejo de Errores**
   - Siempre manejar errores apropiadamente tanto en suspendCoroutine como en callbackFlow
   - Usar bloques try-catch para suspendCoroutine
   - Usar el operador catch para flows

2. **Gestión de Recursos**
   - Limpiar recursos en awaitClose para callbackFlow
   - Usar suspendCancellableCoroutine cuando se necesite manejar cancelación

3. **Consideraciones de Contrapresión**
   - Elegir estrategias de buffer apropiadas para tu caso de uso
   - Considerar usar canales conflated o buffered según tus necesidades

4. **Pruebas**
   - Escribir pruebas tanto para escenarios de éxito como de error
   - Probar el comportamiento de cancelación
   - Verificar la limpieza de recursos

## Patrones Comunes y Ejemplos

### Manejo de Timeout

```kotlin
suspend fun apiCallWithTimeout(): Result<String> = 
    withTimeout(5000L) {
        suspendCoroutine<Result<String>> { continuation: Continuation<Result<String>> ->
            api.call(object : ApiCallback {
                override fun onSuccess(result: Result<String>) {
                    continuation.resume(result)
                }

                override fun onError(error: Exception) {
                    continuation.resumeWithException(error)
                }
            })
        }
    }
```

### Combinando Múltiples Callbacks

```kotlin
fun MultipleSourceService.combinedUpdatesFlow(): Flow<Update> = callbackFlow {
    val callback1 = object : SourceCallback {
        override fun onUpdate(data: Update) {
            trySend(data)
        }
    }

    val callback2 = object : SourceCallback {
        override fun onUpdate(data: Update) {
            trySend(data)
        }
    }

    registerCallbacks(callback1, callback2)

    awaitClose {
        unregisterCallbacks(callback1, callback2)
    }
}.buffer(Channel.BUFFERED)
```

## Conclusión

La conversión de APIs basadas en callbacks a corrutinas y flows puede mejorar significativamente la legibilidad y mantenibilidad del código. Usando `suspendCoroutine` para operaciones únicas y `callbackFlow` para flujos de datos, puedes modernizar código legacy y aprovechar al máximo las potentes características de concurrencia de Kotlin.

Recuerda siempre manejar los errores apropiadamente, gestionar los recursos correctamente y considerar la contrapresión cuando trates con actualizaciones de alta frecuencia. Con estas herramientas y patrones, puedes cerrar efectivamente la brecha entre las APIs basadas en callbacks y la concurrencia moderna de Kotlin.
