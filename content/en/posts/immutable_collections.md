---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Exploring Kotlin’s Immutable Collections Library"
date: 2025-01-30T08:00:00+01:00
description: "Exploring Kotlin’s Immutable Collections Library, use it in Compose to improve performance."
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

# Exploring Kotlin’s Immutable Collections Library

Kotlin's standard collections (`List`, `Set`, `Map`) are mutable by default, which can lead to unintended modifications. To enforce immutability at the API level, JetBrains introduced the **Kotlin Immutable Collections library**. This library provides a set of truly immutable collection types that prevent accidental modifications and enhance safety in concurrent or multi-threaded environments.

---

## Why Use Immutable Collections?

While Kotlin already has `listOf()`, `setOf()`, and `mapOf()` for read-only collections, they are **not truly immutable**. The underlying collection can still be modified if it's referenced elsewhere. Example:

```kotlin
val list = mutableListOf("A", "B", "C")
val readOnlyList: List<String> = list  
list.add("D")  // Modifies the original list  
println(readOnlyList) // Output: [A, B, C, D]  

(readOnlyList as MutableList).add("E")
println(readOnlyList) // Output: [A, B, C, D, E]
```

To solve this, the **Immutable Collections library** provides collections that guarantee immutability at runtime.

---

## Key Features

1. **Truly Immutable** – Once created, they cannot be changed.
2. **Safe for Multi-threading** – Avoids unintended modifications in concurrent environments.
3. **Optimized for Performance** – Uses structural sharing to prevent unnecessary copying.

---

## How to Use Kotlin Immutable Collections

### 1. Add the Dependency

inFirst, include the **Immutable Collections** dependency in your `build.gradle.kts`:

```kotlin
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-collections-immutable:0.3.5")
}
```

### 2. Creating Immutable Collections

The library provides `persistentListOf()`, `persistentSetOf()`, and `persistentMapOf()` to create immutable collections:

```kotlin
import kotlinx.collections.immutable.*

val immutableList = persistentListOf("A", "B", "C")
val immutableSet = persistentSetOf(1, 2, 3)
val immutableMap = persistentMapOf("key1" to 100, "key2" to 200)
```

### 3. Adding and Removing Elements

Since these collections are immutable, modifying operations return **a new modified copy** instead of changing the original collection:

```kotlin
val newList = immutableList.add("D") // Creates a new list
println(newList)  // Output: [A, B, C, D]

val newMap = immutableMap.put("key3", 300)
println(newMap)   // Output: {key1=100, key2=200, key3=300}
```

The original `immutableList` and `immutableMap` remain unchanged!

---

## Performance Considerations

Unlike regular immutable collections (which require full copies for modifications), **persistent collections** use **structural sharing**. This means that modifications create a new collection while reusing unchanged parts of the original, improving performance and memory efficiency.

For example, adding an item to a persistent list does not create a full copy but instead reuses most of the existing structure:

```
Original:  [A, B, C]  
New List:  [A, B, C, D] (Only "D" is newly allocated)  
```

This makes immutable collections efficient even for large datasets.

---

## Benefits in Jetpack Compose

Immutable collections are particularly useful in **Jetpack Compose** because they optimize **state management and recompositions**. Here’s why they matter in Compose applications:

### 1. Avoid Unnecessary Recompositions

- Compose tracks state changes to decide when to recompose UI elements.
- Mutable lists, sets, or maps might trigger **unnecessary recompositions** even when data hasn’t changed.
- Immutable collections ensure that state remains **stable**, preventing redundant recompositions.

**Example:**

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

If `items` is a **mutable list**, even reassigning the same values **triggers recomposition**. Using an **immutable collection** like `PersistentList` ensures that Compose recognizes when data is unchanged:

```kotlin
val items = remember { persistentListOf("A", "B", "C") }
MyListScreen(items)
```

### 2. State Stability for Performance

- Compose optimizes rendering by skipping recompositions when state objects are **stable**.
- Immutable collections use **structural sharing**, meaning that modifications only affect the changed part while reusing the rest.
- This leads to better performance in **large lists or complex UI hierarchies**.

### 3. Predictable UI Behavior

- Since immutable collections **cannot be modified** after creation, they prevent accidental mutations that might lead to unpredictable UI updates.
- This is especially useful in **state-driven architectures (MVI, Redux, etc.)**, ensuring UI updates only when necessary.

### 4. Thread Safety

- In Compose apps using **coroutines (Flows, LiveData, etc.)**, immutable collections prevent race conditions when multiple threads update state.
- They ensure safe data flow between **ViewModels, repositories, and UI components**.

---

## When to Use Immutable Collections?

✅ **Functional Programming** – Encourages immutability for safer data transformations.\
✅ **Thread Safety** – Prevents unintended modifications in multi-threaded environments.\
✅ **Prevent Bugs** – Reduces unexpected side effects due to accidental mutation.\
✅ **State Management in Compose** – Helps optimize recompositions and improve UI performance.

---

## Conclusion

The Kotlin Immutable Collections library provides **truly immutable**, **efficient**, and **safe** collections, making them a great choice for functional programming, concurrent applications, and Jetpack Compose development. By leveraging **persistent collections**, you can write safer and more predictable Kotlin code.

🚀 **Would you use immutable collections in your projects?**
