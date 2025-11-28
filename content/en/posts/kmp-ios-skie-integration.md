---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Seamless KMP on iOS: Enhancing the Experience with SKIE"
date: 2025-11-28T10:00:00+01:00
description: "Discover how SKIE transforms the Kotlin Multiplatform experience for iOS developers by bridging the gap between Kotlin and Swift semantics."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/swift-suspend.png
draft: false
tags: 
- kotlin-multiplatform
- ios
- swift
- skie
- kmp
---

### Seamless KMP on iOS: Enhancing the Experience with SKIE

Kotlin Multiplatform (KMP) is a game-changer for sharing business logic across platforms. However, for iOS developers consuming this shared code, the experience can sometimes feel foreign. This is because KMP compiles to iOS via Objective-C interoperability, which doesn't natively support some of modern Kotlin's (and Swift's) best features.

This is where **[SKIE](https://skie.touchlab.co/)** (Swift Kotlin Interface Enhancer), developed and maintained by **Touchlab**, comes in.

It is worth noting that JetBrains is working on `swift-export` to natively improve the iOS developer experience. However, until that feature is ready, tools like SKIE are the best way to bridge the gap.

---

#### The "Vanilla" KMP Experience on iOS

When you compile Kotlin to an iOS framework without tools like SKIE, you might encounter several friction points:

1.  **Sealed Classes**: In Kotlin, `sealed classes` allow for exhaustive `when` statements. In Objective-C (and thus Swift via KMP), these become standard class hierarchies. You lose the compiler's ability to ensure you've handled all cases in a `switch` statement.
2.  **Enums**: Similar to sealed classes, Kotlin enums don't always map 1-to-1 to Swift enums with the same power and flexibility.
3.  **Coroutines**: Kotlin's `suspend` functions are generated as functions with completion handlers (callbacks) in Objective-C. While Swift has `async/await`, bridging these callbacks manually is tedious and error-prone.
4.  **Flows**: Kotlin `Flow` is a powerful stream processing tool. In standard KMP, it appears as a generic object, requiring adapters to be consumed comfortably in Swift.

---

#### How SKIE Improves the iOS Developer Experience

SKIE acts as a bridge that generates Swift-friendly code on top of the Objective-C header. It doesn't change the Kotlin compiler; it enhances the output.

##### 1. Exhaustive Sealed Classes & Enums
SKIE generates real Swift enums for your Kotlin `sealed classes` and `interfaces`. This means you can use Swift's `switch` statement with full exhaustiveness checking.

```swift
// Swift code with SKIE
switch viewState {
case .loading:
    showLoader()
case .data(let content):
    showContent(content)
case .error(let message):
    showError(message)
}
```

Without SKIE, you would have to cast types or check `is` types, often needing a `default` case that swallows future errors.

##### 2. Native Async/Await Support
SKIE automatically converts Kotlin `suspend` functions into Swift `async throws` functions. This allows iOS developers to use the modern concurrency model they are used to.

```swift
// Swift code with SKIE
do {
    let data = try await repository.getData()
    updateUI(data)
} catch {
    handleError(error)
}
```

##### 3. Flows as AsyncSequences
Perhaps one of the coolest features is transforming Kotlin `Flow` into Swift `AsyncSequence`. This allows you to iterate over data streams using a simple `for await` loop.

```swift
// Swift code with SKIE
for await state in viewModel.stateFlow {
    render(state)
}
```

---

#### How to Configure SKIE

Setting up SKIE is incredibly straightforward. It is distributed as a Gradle plugin.

1.  Open your root `build.gradle.kts` or the `build.gradle.kts` of your shared module.
2.  Add the SKIE plugin to the `plugins` block:

```kotlin
plugins {
    kotlin("multiplatform") version "2.2.21" // or your version
    id("co.touchlab.skie") version "0.10.8" // Check for latest version
}
```

3.  Sync your Gradle project.

That's it! SKIE will automatically hook into the link task of your iOS framework and generate the enhanced Swift code.

For more advanced configuration, such as disabling specific features or fine-tuning enum generation, you can use the `skie` configuration block:

```kotlin
skie {
    features {
        group("com.example.library") {
            SealedInterop.enabled = true
        }
    }
}
```

---

#### Conclusion

If you are building a KMP project that targets iOS, using SKIE is almost mandatory for a polished experience. It respects the platform differences while allowing you to share the maximum amount of logic, keeping your iOS team happy and your codebase clean.
