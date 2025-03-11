---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Converting Callbacks to Coroutines and Flows in Kotlin"
date: 2025-03-11T08:00:00+01:00
description: "Learn how to transform callback-based APIs into modern Kotlin coroutines and flows using suspendCoroutine and callbackFlow"
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

# Converting Callbacks to Coroutines and Flows in Kotlin

Callback-based APIs have been a common pattern in asynchronous programming for many years. However, with Kotlin's coroutines and flows, we can transform these callbacks into more modern, sequential code that's easier to read and maintain. In this article, we'll explore how to use `suspendCoroutine` and `callbackFlow` to convert callback-based APIs into coroutines and flows.

## Understanding suspendCoroutine

The `suspendCoroutine` function is a powerful tool that allows you to wrap callback-based APIs into suspend functions. This transformation makes asynchronous code more sequential and easier to handle.

### Basic Usage of suspendCoroutine

Here's a simple example of converting a callback-based function to a suspend function:

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
        println("Location received: $location")
    } catch (e: Exception) {
        println("Error getting location: ${e.message}")
    }
}
```

### Handling Cancellation

When working with `suspendCoroutine`, it's important to handle cancellation properly:

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

## Converting to Flows with callbackFlow

While `suspendCoroutine` is great for one-shot operations, `callbackFlow` is perfect for handling streams of data or events. It allows you to convert callback-based APIs that emit multiple values into Kotlin Flows.

### Basic callbackFlow Example

Here's how to convert a location updates API to a Flow:

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
            println("Error in location updates: ${error.message}")
        }
        .collect { location: Location ->
            println("New location: $location")
        }
}
```

### Handling Backpressure

When dealing with frequent updates, it's important to handle backpressure properly:

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

## Best Practices

1. **Error Handling** 
   - Always handle errors appropriately in both suspendCoroutine and callbackFlow
   - Use try-catch blocks for suspendCoroutine
   - Use catch operator for flows

2. **Resource Management**
   - Clean up resources in awaitClose for callbackFlow
   - Use suspendCancellableCoroutine when cancellation handling is needed

3. **Backpressure Considerations**
   - Choose appropriate buffer strategies for your use case
   - Consider using conflated or buffered channels based on your needs

4. **Testing**
   - Write tests for both success and error scenarios
   - Test cancellation behavior
   - Verify resource cleanup

## Common Patterns and Examples

### Timeout Handling

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

### Combining Multiple Callbacks

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

## Conclusion

Converting callback-based APIs to coroutines and flows can significantly improve code readability and maintainability. By using `suspendCoroutine` for one-shot operations and `callbackFlow` for streams of data, you can modernize legacy code and take full advantage of Kotlin's powerful concurrency features.

Remember to always handle errors appropriately, manage resources properly, and consider backpressure when dealing with high-frequency updates. With these tools and patterns, you can effectively bridge the gap between callback-based APIs and modern Kotlin concurrency.
