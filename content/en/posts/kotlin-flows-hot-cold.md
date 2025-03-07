---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Understanding Hot and Cold Flows in Kotlin"
date: 2025-03-07T08:00:00+01:00
description: "A comprehensive guide to understanding the differences between hot and cold flows in Kotlin, with practical examples"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/flows.png
draft: false
tags:
- kotlin
- coroutines
- flows
---

# Understanding Hot and Cold Flows in Kotlin

Kotlin Flow is a powerful feature for handling reactive streams of data. One of the fundamental concepts to understand when working with flows is the distinction between hot and cold flows. This article will explain the differences and provide practical examples of both types.

## Cold Flows: The Default Behavior

Cold flows are the default type in Kotlin Flow. They start producing values only when a collector starts collecting from them. Each collector gets its own independent stream of values from scratch.

### Characteristics of Cold Flows:

- Start producing values only when collected
- Each collector receives all values from the beginning
- Values are produced independently for each collector
- Resources are not shared between collectors

Here's an example of a cold flow:

```kotlin
fun numbers(): Flow<Int> = flow {
    println("Flow started")
    for (i in 1..3) {
        delay(100)
        emit(i)
    }
}

suspend fun main() {
    val numbersFlow = numbers()

    // First collector
    println("First collector starting")
    numbersFlow.collect { value: Int ->
        println("Collector 1: $value")
    }

    // Second collector
    println("Second collector starting")
    numbersFlow.collect { value: Int ->
        println("Collector 2: $value")
    }
}
```

Output:
```
First collector starting
Flow started
Collector 1: 1
Collector 1: 2
Collector 1: 3
Second collector starting
Flow started
Collector 2: 1
Collector 2: 2
Collector 2: 3
```

## Hot Flows: Shared State and Events

Hot flows, on the other hand, may start producing values regardless of collectors and can share the same stream of values between multiple collectors. They're useful for representing real-time events or shared state.

### Types of Hot Flows:

1. **StateFlow**: For representing state
2. **SharedFlow**: For representing events

### StateFlow Example:

```kotlin
class TemperatureSensor {
    private val _temperature = MutableStateFlow(20.0) // Initial temperature
    val temperature: StateFlow<Double> = _temperature.asStateFlow()

    fun updateTemperature(newTemp: Double) {
        _temperature.update { newTemp }
    }
}

suspend fun main() {
    val sensor = TemperatureSensor()

    // First collector
    launch {
        sensor.temperature.collect { temp: Double ->
            println("Display 1: $temp°C")
        }
    }

    delay(100)
    println("Updating temperature to 22.5°C")
    sensor.updateTemperature(22.5)

    delay(100)
    // Late collector - will only see the current value (22.5) and future updates
    launch {
        println("Display 2 starting collection...")
        sensor.temperature.collect { temp: Double ->
            println("Display 2: $temp°C")
        }
    }

    delay(100)
    println("Updating temperature to 23.0°C")
    sensor.updateTemperature(23.0)
}
```

Output:
```
Display 1: 20.0°C
Updating temperature to 22.5°C
Display 1: 22.5°C
Display 2 starting collection...
Display 2: 22.5°C
Updating temperature to 23.0°C
Display 1: 23.0°C
Display 2: 23.0°C
```

### SharedFlow Example:

```kotlin
class EventBus {
    private val _events = MutableSharedFlow<String>()
    val events = _events.asSharedFlow()

    suspend fun emit(event: String) {
        _events.emit(event)
    }
}

suspend fun main() {
    val eventBus = EventBus()

    // First subscriber
    launch {
        eventBus.events.collect { event: String ->
            println("Subscriber 1: $event")
        }
    }

    delay(100)
    println("Emitting first event")
    eventBus.emit("User logged in")

    delay(100)
    // Late subscriber - will miss the "User logged in" event
    launch {
        println("Subscriber 2 starting collection...")
        eventBus.events.collect { event: String ->
            println("Subscriber 2: $event")
        }
    }

    delay(100)
    println("Emitting second event")
    eventBus.emit("Data updated")
}
```

Output:
```
Subscriber 1: User logged in
Subscriber 2 starting collection...
Emitting second event
Subscriber 1: Data updated
Subscriber 2: Data updated
```

## Key Differences Between Hot and Cold Flows

1. **Execution Timing**
   - Cold: Executes per collector
   - Hot: Can execute independently of collectors

2. **Value Sharing**
   - Cold: Each collector gets its own stream
   - Hot: Multiple collectors share the same stream

3. **Resource Usage**
   - Cold: Resources allocated per collector
   - Hot: Resources shared between collectors

4. **Use Cases**
   - Cold: Data transformations, database queries
   - Hot: UI states, real-time events, broadcasts

## When to Use Each Type

### Use Cold Flows When:
- Each collector needs its own independent stream
- You're performing resource-intensive operations
- You need to restart the stream from the beginning for each collector

### Use Hot Flows When:
- Multiple parts of your app need the same data stream
- You're dealing with UI state management
- You need to broadcast events to multiple subscribers
- You want to share resources between collectors

## Best Practices

1. **Cold Flows**
   - Use for operations that should be executed independently
   - Consider using buffer() for performance optimization
   - Clean up resources in onCompletion

2. **Hot Flows**
   - Use StateFlow for state management
   - Use SharedFlow for events
   - Consider replay and buffer sizes carefully
   - Handle backpressure appropriately

By understanding these differences, you can choose the right type of flow for your specific use case and create more efficient and maintainable reactive streams in your Kotlin applications.
