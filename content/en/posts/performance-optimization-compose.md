---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Performance Optimization in Jetpack Compose"
date: 2025-04-08T08:00:00+01:00
description: "Learn essential techniques and best practices for optimizing performance in Jetpack Compose applications, including composition optimization, recomposition control, and memory management"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/remember-optimization.png
draft: false
tags:
- android
- compose
- performance
- optimization
---

# Performance Optimization in Jetpack Compose

Performance optimization is crucial for delivering a smooth user experience in Jetpack Compose applications. This article explores key techniques and best practices to ensure your composable functions are efficient and performant.

## Understanding Composition and Recomposition

One of the fundamental aspects of performance in Compose is understanding how composition and recomposition work:

### Smart Recomposition

Compose uses smart recomposition to update only the parts of the UI that need to change. Understanding what triggers recomposition and how to minimize its scope is crucial for performance optimization.

```kotlin
@Composable
fun ExpensiveCalculation(numbers: List<Int>) {
    // Bad: Expensive operation performed on every recomposition
    val average = numbers.takeIf { it.isNotEmpty() }
        ?.average()
        ?: 0.0

    // Good: Expensive operation cached and only recalculated when input changes
    val cachedAverage = remember(numbers) {
        numbers.takeIf { it.isNotEmpty() }
            ?.average()
            ?: 0.0
    }

    Column {
        // This will recalculate on every recomposition
        Text("Current Average: ${"%.2f".format(average)}")

        // This will use the cached value
        Text("Cached Average: ${"%.2f".format(cachedAverage)}")
    }
}
```

### Stable Types and Immutability

Stable types are crucial for Compose's smart recomposition system. A type is considered stable when Compose can guarantee that its equals() method is consistent with its properties and that the properties themselves won't change without triggering a recomposition.

```kotlin
// Bad: Unstable type - mutable properties can change without notifying Compose
data class UserState(
    var name: String,  // Mutable property can change silently
    var age: Int      // Changes won't trigger recomposition
)

// Good: Stable type - immutable properties and explicit stability
@Stable  // Tells Compose this type has a predictable equality
data class UserState(
    val name: String,  // Immutable property
    val age: Int      // Changes require creating a new instance
)
```

Using stable types provides several benefits:
1. More efficient recomposition - Compose can skip recomposing parts of the UI when it knows the data hasn't changed
2. Predictable behavior - Changes to the data always trigger proper UI updates
3. Thread safety - Immutable data is safe to share across coroutines

## Key Performance Optimizations

### 1. State Management with remember and derivedStateOf

The `remember` and `derivedStateOf` functions serve different purposes in state management:

```kotlin
@Composable
fun UserProfile(user: User, items: List<Item>) {
    // Bad: Recalculating on every recomposition
    val filteredItems = items.filter { it.userId == user.id }

    // Good: Caching calculation with remember
    val cachedItems = remember(items, user.id) {
        items.filter { it.userId == user.id }
    }

    // Better: Using derivedStateOf for reactive computations
    val reactiveItems by remember(items) {
        derivedStateOf { 
            items.filter { it.userId == user.id }
        }
    }

    // reactiveItems will automatically update when items changes
    // and only trigger recomposition when the filtered result changes
    LazyColumn {
        itemsIndexed(
            items = reactiveItems,
            key = { _: Int, item: Item -> item.id }
        ) { _: Int, item: Item ->
            ItemRow(item)
        }
    }
}
```

### 2. Composition Local Usage

```kotlin
// Bad: Each child component accesses CompositionLocal
@Composable
fun DeepNestedContent() {
    val theme = LocalTheme.current  // Accessed directly
    val strings = LocalStrings.current  // Multiple CompositionLocal accesses
    val dimensions = LocalDimensions.current

    Column {
        Text(
            text = strings.title,
            style = theme.textStyle,
            modifier = Modifier.padding(dimensions.padding)
        )
        // More nested content with repeated CompositionLocal access
    }
}

// Good: Hoisting CompositionLocal values to minimize lookups
@Composable
fun ParentContent() {
    // Single access to CompositionLocal values
    val theme = LocalTheme.current
    val strings = LocalStrings.current
    val dimensions = LocalDimensions.current

    DeepNestedContent(
        theme = theme,
        strings = strings,
        dimensions = dimensions
    )
}

@Composable
fun DeepNestedContent(
    theme: Theme,
    strings: Strings,
    dimensions: Dimensions
) {
    // Use passed parameters instead of looking up CompositionLocal values
    Column {
        Text(
            text = strings.title,
            style = theme.textStyle,
            modifier = Modifier.padding(dimensions.padding)
        )
        // More nested content using passed parameters
    }
}
```

### 3. LazyList Optimizations

Efficient list rendering is crucial for smooth scrolling performance. Here are key optimizations for LazyList components:

```kotlin
@Composable
fun <T : Any> OptimizedList(items: List<T>) {
    LazyColumn {
        itemsIndexed(
            items = items,
            // Stable keys help Compose track items across updates
            key = { _: Int, item: T -> item.hashCode() }
        ) { _: Int, item: T ->
            // Content for each item
        }
    }
}
```

Key optimizations for LazyList:
1. Provide stable keys to help Compose track items across updates
2. Use fixed sizes when possible to avoid remeasurement
3. Keep item composables lightweight
4. Avoid unnecessary allocations in item content
5. Use `remember` to cache expensive computations per item

## Measuring and Monitoring Performance

### Layout Inspector and Composition Traces

The Layout Inspector in Android Studio is a powerful tool for debugging Compose UI performance. It provides insights into your app's view hierarchy, recomposition counts, and modifiers applied to each composable.

To use Layout Inspector with Compose:

1. Run your app in debug mode
2. In the Running Devices windows you will find a button to Toggle Layout inspector
3. Inspect the Compose hierarchy:
   - View component tree
   - Check recomposition counts
   - Analyze modifier chains
   - Inspect composable parameters

![Toggle Layout inspector](https://carrion.dev/images/kotlin/layout-inspector.png)

Key metrics to monitor in Layout Inspector:
1. Recomposition counts - High numbers indicate potential optimization opportunities
2. Skipping counts - Check that your Composables are skipping recomposition when they should
3. Modifier chain complexity - Long chains might affect measure/layout performance

### Performance Testing

```kotlin
@Test
fun performanceTest() {
    benchmarkRule.measureRepeated(
        packageName = "com.example.app",
        metrics = listOf(FrameTimingMetric()),
        iterations = 5
    ) {
        composeTestRule.setContent {
            YourComposable()
        }
    }
}
```

## Best Practices Summary

1. Use stable types and immutable data structures
2. Hoist expensive computations with `remember`
3. Implement proper keys in lazy lists
4. Minimize the scope of recomposition
5. Profile and measure performance regularly

Following these optimization techniques will help ensure your Compose UI remains responsive and efficient, providing a better user experience for your applications.
