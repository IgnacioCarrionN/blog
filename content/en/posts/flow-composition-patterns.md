---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Flow Composition Patterns: Combining Multiple Flows Effectively"
date: 2025-03-18T08:00:00+01:00
description: "Learn how to effectively combine multiple Kotlin Flows using various composition patterns and best practices for building complex flow pipelines"
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

# Flow Composition Patterns: Combining Multiple Flows Effectively

When working with Kotlin Flows in real-world applications, you often need to combine multiple data streams to create more complex workflows. This article explores various Flow composition patterns and best practices for effectively combining multiple Flows.

## Understanding Flow Composition

Flow composition is the process of combining multiple Flows to create a new Flow that represents a more complex data stream. Kotlin provides several operators for Flow composition, each serving different use cases.

### Basic Flow Composition Operators

Let's start with the fundamental Flow composition operators:

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

The `zip` and `combine` operators serve different purposes and have distinct behaviors when working with multiple flows:

#### Zip Operator
- Pairs values strictly one-to-one from each flow
- Waits for all flows to emit before producing a result
- If one flow emits slower, it creates back-pressure
- Useful when you need to match corresponding values from different flows
- If one flow completes, the resulting flow also completes

#### Combine Operator
- Uses the latest value from each flow to produce results
- Emits whenever any flow produces a new value
- No back-pressure - uses the most recent value from other flows
- Useful for real-time updates where you need the latest state
- Continues until all flows complete

Here's a practical example showing the difference:

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

Let's see how they behave with different timing:

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

Output for zippedFlow (pairs values in order):
```
"Price: 10.0, Quantity: 5"    // First pair
"Price: 11.0, Quantity: 6"    // Second pair
// 12.0 is never emitted because quantities has no more values
```

Output for combinedFlow (reacts to each change):
```
"Latest Price: 10.0, Latest Quantity: 5"    // Initial values
"Latest Price: 11.0, Latest Quantity: 5"    // Price updated at t=100ms
"Latest Price: 11.0, Latest Quantity: 6"    // Quantity updated at t=150ms
"Latest Price: 12.0, Latest Quantity: 6"    // Price updated at t=200ms
```

This example shows how:
- `zip` matches values in sequence and requires both flows to emit
- `combine` reacts to changes in either flow and uses the latest available values
- `zip` might skip values if flows emit at different rates
- `combine` ensures you always work with the most recent data

## Advanced Composition Patterns

### Merging Multiple Flows

The `merge` operator combines multiple flows into a single flow, preserving the relative timing of emissions from each source. Unlike `zip` or `combine`, `merge` simply forwards values as they arrive, without attempting to pair or combine them.

#### Key Characteristics of Merge
- Emits values as soon as they arrive from any source flow
- Maintains the order of emissions within each source flow
- Doesn't wait for or combine values from different flows
- Completes only when all source flows complete
- Useful for handling independent events from multiple sources

Here's how merge works with multiple event sources:

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

Let's see how merge handles events with different timing:

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

Output (events in order of arrival):
```
"Click 1"    // t=0ms   (from clickEvents)
"Key A"      // t=50ms  (from keyEvents)
"Swipe"      // t=75ms  (from gestureEvents)
"Click 2"    // t=100ms (from clickEvents)
"Key B"      // t=150ms (from keyEvents)
```

Common Use Cases for Merge:
1. **Event Handling**: Combining user interactions from different sources
2. **Multi-source Updates**: Monitoring changes from multiple independent data sources
3. **Parallel Processing**: Collecting results from parallel operations
4. **System Monitoring**: Aggregating logs or metrics from multiple components

The alternative implementation using a Flow builder shows how merge works internally:

## Error Handling in Composed Flows

When working with composed flows, error handling becomes particularly important as errors can propagate through the flow chain and affect multiple data sources. There are several strategies for handling errors in composed flows:

### 1. Individual Flow Error Handling
Each flow can handle its own errors before composition:

```kotlin
val safeFlow1 = flow1.catch { error: Throwable ->
    emit(fallbackValue)
    // or
    emit(Result.failure<String>(error))
}

val safeFlow2 = flow2.catch { error: Throwable ->
    // Log error and emit default value
    logger.error("Flow 2 failed", error)
    emit(defaultValue)
}
```

### 2. Error Transformation in Composed Flows
Transform errors into domain-specific results:

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

// Usage in composition
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

### 3. Using Result Type for Error Handling
A common pattern using Kotlin's Result type:

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

### 4. Retry Strategies
Implement retry logic for transient failures:

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

// Usage in composed flows
val resilientFlow = combine(
    flow1.retryWithBackoff(),
    flow2.retryWithBackoff()
) { value1: Data1, value2: Data2 ->
    // Process values
    CombinedData(value1, value2)
}
```

### Best Practices for Error Handling in Composed Flows:

1. **Handle Errors Close to Source**:
   - Catch errors in individual flows before composition
   - Transform errors into domain-specific results
   - Provide fallback values when appropriate

2. **Error Recovery Strategies**:
   - Implement retry logic for transient failures
   - Use backoff strategies to avoid overwhelming systems
   - Consider providing default or cached values

3. **Error Propagation**:
   - Decide whether to propagate or handle errors locally
   - Use structured error types (sealed classes, Result type)
   - Maintain error context through the flow chain

4. **Monitoring and Debugging**:
   - Log errors with appropriate context
   - Track error rates and patterns
   - Implement proper error reporting

## Performance Considerations

When combining Flows, consider these performance optimization techniques:

1. **Buffer Management**:
```kotlin
val optimizedFlow = flow1
    .buffer(Channel.BUFFERED)
    .combine(flow2.buffer(Channel.BUFFERED)) { value1: T1, value2: T2 ->
        // Process values
    }
```

2. **Conflation for Latest Values**:
```kotlin
val conflatedFlow = flow1
    .conflate()
    .combine(flow2.conflate()) { value1: T1, value2: T2 ->
        // Process only latest values
    }
```

## Conclusion

Flow composition is a powerful feature that allows you to build complex reactive streams in Kotlin. By understanding these patterns and best practices, you can effectively combine multiple Flows while maintaining clean, maintainable, and performant code. Remember to:

- Choose the right composition operator for your use case
- Handle errors appropriately at each level
- Consider performance implications
- Implement proper cancellation handling

These patterns will help you build robust applications that can handle complex data flows effectively.
