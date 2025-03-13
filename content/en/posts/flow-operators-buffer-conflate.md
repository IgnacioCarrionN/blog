---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Understanding Flow Operators: Buffer, Conflate, Debounce, and Sample"
date: 2025-03-13T08:00:00+01:00
description: "Deep dive into Kotlin Flow operators: buffer, conflate, debounce, and sample. Learn when and how to use each operator with practical examples."
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

# Understanding Flow Operators: Buffer, Conflate, Debounce, and Sample

When working with Kotlin Flows, especially in scenarios involving fast-emitting producers and slow collectors, it's crucial to understand how to manage the flow of data effectively. This post explores four essential Flow operators that help handle such scenarios: `buffer`, `conflate`, `debounce`, and `sample`.

## The Problem: Slow Collectors

Before diving into the operators, let's understand the problem they solve. Consider this scenario:

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

In this case, the collector is slower than the producer, which can lead to backpressure issues. Each operator we'll discuss provides a different strategy to handle this situation.

## Buffer Operator

The `buffer` operator creates a channel of specified capacity to store emissions while the collector processes previous values.

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

### When to Use Buffer
- When you want to store a specific number of emissions
- When you need to process all values but want to decouple producer and collector speeds
- When order of processing is important

## Conflate Operator

The `conflate` operator keeps only the latest value, dropping intermediate ones if the collector can't keep up.

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

### When to Use Conflate
- When you only care about the most recent value
- In UI scenarios where showing intermediate states isn't necessary
- When processing every value isn't critical

## Debounce Operator

The `debounce` operator emits a value only after a specified time has passed without new emissions.

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

### When to Use Debounce
- For search-as-you-type functionality
- When handling rapid UI events
- When you want to wait for "quiet periods" before processing

## Sample Operator

The `sample` operator periodically samples the most recent value from the flow at specified intervals.

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

### When to Use Sample
- When you need regular updates at fixed intervals
- For displaying real-time data where intermediate values aren't crucial
- When you want to limit the rate of processing regardless of emission rate

## Comparison and Best Practices

Here's a quick comparison of these operators:

| Operator  | Behavior | Use Case |
|-----------|----------|-----------|
| buffer    | Stores emissions | Process all values, maintain order |
| conflate  | Keeps latest only | UI updates, latest-value-only scenarios |
| debounce  | Waits for quiet period | Search-as-you-type, rapid event handling |
| sample    | Takes periodic snapshots | Regular updates, rate limiting |

## Conclusion

Understanding these Flow operators is crucial for building efficient reactive applications:
- Use `buffer` when you need to process all values and control memory usage
- Use `conflate` when only the latest value matters
- Use `debounce` when handling rapid events that need "settling time"
- Use `sample` when you need regular updates at fixed intervals

Choose the appropriate operator based on your specific use case and requirements regarding data completeness, order, and processing rate.

Remember that these operators can be combined to create more sophisticated data processing pipelines, but be careful not to over-complicate your flows. Always consider the trade-offs between data completeness, memory usage, and processing efficiency.