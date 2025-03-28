---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Building Type-safe DSLs with Kotlin: From Basics to Advanced Patterns"
date: 2025-03-25T08:00:00+01:00
description: "Learn how to create type-safe Domain-Specific Languages (DSLs) in Kotlin using @DslMarker for better scope control"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/type-safe-dsls.png
draft: false
tags:
- kotlin
- dsl
- type-safety
- design-patterns
---

# Building Type-safe DSLs with Kotlin: From Basics to Advanced Patterns

Domain-Specific Languages (DSLs) in Kotlin allow you to create expressive, readable, and type-safe APIs. This article explores how to build effective DSLs using Kotlin's powerful features, focusing on scope control with `@DslMarker` to prevent common mistakes in nested DSLs.

By the end of this article, you'll understand:
- How to design clean and intuitive DSL APIs
- When and how to use `@DslMarker` for better scope control
- Best practices for maintaining type safety throughout your DSL
- Common pitfalls and how to avoid them

## Basic DSL Concepts

Let's explore the fundamental concepts of Kotlin DSLs by building a simple HTML builder:

### HTML Builder DSL

```kotlin
// Traditional way: manual string building
val sb = StringBuilder()
sb.append("<div>")
sb.append("<span>Hello</span>")
sb.append("<div>")      // Easy to make mistakes
sb.append("<span>Nested content</span>")
sb.append("</div>")     // Which div are we closing?
sb.append("</div>")     // Is the nesting correct?

// Using a DSL: structure mirrors HTML's natural nesting
val html = buildHtml {
    div {
        span { +"Hello" }
        div {
            span { +"Nested content" }
        }  // Nesting is clear and enforced by the compiler
    }
}

// The DSL implementation using Kotlin's features
class HtmlBuilder {
    private val content = StringBuilder()

    // Extension function with receiver: creates div { } blocks
    fun div(block: HtmlBuilder.() -> Unit) {
        content.append("<div>")
        this.block()  // 'this' is the receiver (HtmlBuilder)
        content.append("</div>")
    }

    // Similar to div, creates span { } blocks
    fun span(block: HtmlBuilder.() -> Unit) {
        content.append("<span>")
        this.block()
        content.append("</span>")
    }

    // Operator overloading: allows +"text" syntax
    operator fun String.unaryPlus() {
        content.append(this)
    }

    override fun toString() = content.toString()
}

// Entry point function that creates the DSL context
fun buildHtml(block: HtmlBuilder.() -> Unit): String =
    HtmlBuilder().apply(block).toString()
```

This HTML builder demonstrates the fundamental concepts that make Kotlin DSLs powerful:
- Extension functions create a natural, fluent syntax
- Lambda receivers provide clear scoping and context
- Operator overloading enables convenient syntactic sugar
- Type safety prevents common mistakes at compile time

## Scope Safety with @DslMarker

When building DSLs with nested scopes, we need to ensure that method calls are unambiguous. Without proper scope control, it can be unclear which method is being called in nested contexts. Here's an example of this problem:

```kotlin
class MenuDsl {
    fun item(text: String) { println("Menu item: $text") }

    fun submenu(block: MenuDsl.() -> Unit) {
        println("Submenu start")
        MenuDsl().block()
        println("Submenu end")
    }
}

fun menu(block: MenuDsl.() -> Unit) = MenuDsl().apply(block)

// This code is ambiguous:
menu {
    item("Home")
    submenu {
        item("Settings")  // Which item() is this calling?
        // Could be from either the outer or inner MenuDsl!
    }
}
```

Kotlin provides the `@DslMarker` annotation to solve this problem:

```kotlin
@DslMarker
annotation class MenuMarker

@MenuMarker
class MenuDsl {
    fun item(text: String) { println("Menu item: $text") }

    fun submenu(block: MenuDsl.() -> Unit) {
        println("Submenu start")
        MenuDsl().block()
        println("Submenu end")
    }
}

// Now the code is unambiguous:
menu {
    item("Home")         // Calls outer scope's item()
    submenu {
        item("Settings") // Calls inner scope's item()
        // Outer scope's item() is not accessible here
    }
}
```

The `@DslMarker` annotation is a powerful tool that helps us build safer DSLs by preventing scope-related ambiguity. Now that we understand the basic concepts and safety features, let's see how they're applied in practice.

## Real-World Examples

Kotlin's ecosystem provides excellent examples of how DSLs can simplify complex APIs while maintaining type and scope safety. Let's look at two different approaches:

### Kotlin's buildString Function

The standard library starts with the basics: string manipulation. While a simple task, `buildString` shows how a well-designed DSL can make even common operations more elegant and less error-prone:

```kotlin
// Without DSL: explicit StringBuilder management
val sb = StringBuilder()
sb.append("Hello, ")
sb.appendLine("World!")
sb.append("The time is ")
sb.append(System.currentTimeMillis())
val message = sb.toString()

// With DSL: StringBuilder methods available directly
val messageDsl = buildString {
    append("Hello, ")
    appendLine("World!")
    append("The time is ")
    append(System.currentTimeMillis())
}
```

The DSL is enabled by this simple yet powerful function in the standard library:

```kotlin
// The actual implementation from Kotlin's standard library
public inline fun buildString(builderAction: StringBuilder.() -> Unit): String {
    contract { callsInPlace(builderAction, InvocationKind.EXACTLY_ONCE) }
    return StringBuilder().apply(builderAction).toString()
}
// The contract helps the compiler ensure type safety in the DSL
// by guaranteeing the builder action is called exactly once
```

Finally, let's look at how these DSL concepts can be applied to complex real-world problems. Jetpack Compose's LazyColumn is an excellent example of how a well-designed DSL can transform a complex UI component into an intuitive, declarative API:

### Jetpack Compose LazyColumn

While our previous examples dealt with text manipulation, LazyColumn tackles a more challenging problem: efficient scrolling lists with complex layouts and recycling. Despite this complexity, the DSL makes it remarkably simple to use:

```kotlin
// The actual LazyColumn signature shows its complexity
@Composable
fun LazyColumn(
    modifier: Modifier = Modifier,
    state: LazyListState = rememberLazyListState(),
    contentPadding: PaddingValues = PaddingValues(0.dp),
    reverseLayout: Boolean = false,
    verticalArrangement: Arrangement.Vertical =
        if (!reverseLayout) Arrangement.Top else Arrangement.Bottom,
    horizontalAlignment: Alignment.Horizontal = Alignment.Start,
    flingBehavior: FlingBehavior = ScrollableDefaults.flingBehavior(),
    userScrollEnabled: Boolean = true,
    content: LazyListScope.() -> Unit  // But the DSL makes it easy to use!
) {
    // Complex implementation with recycling, layout, animations...
}

// ...but the DSL makes it simple through this interface
interface LazyListScope {
    // Add a single item to the list
    fun item(
        key: Any? = null,
        contentType: Any? = null,
        content: @Composable () -> Unit
    )

    // Add multiple items from a list
    fun <T> items(
        items: List<T>,
        key: ((item: T) -> Any)? = null,
        contentType: (item: T) -> Any? = { null },
        itemContent: @Composable (item: T) -> Unit
    )
}

// Example: Using the DSL to create a complex list
@Composable
fun CategoryList(categories: List<String>) {
    LazyColumn {
        // Header section
        item {
            Text("Categories")
        }

        // Dynamic items with custom keys
        items<String>(
            items = categories,
            key = { category: String -> category }  // Stable identity for animations
        ) { category: String ->
            Text("Category: $category")
        }

        // Footer section
        item {
            Text("End of list")
        }
    }
}

// The LazyColumn DSL demonstrates how Kotlin can:
// - Hide complex configuration behind sensible defaults
// - Transform verbose APIs into simple, focused interfaces
// - Make powerful features (recycling, animations) easy to use
// - Turn implementation complexity into developer-friendly APIs
```

## Conclusion

Building DSLs in Kotlin allows you to create powerful, expressive APIs that are both safe and intuitive to use. As we've seen through our examples, from a simple HTML builder to Jetpack Compose's sophisticated UI components, well-designed DSLs can:

1. **Improve Code Quality**
   - Transform verbose imperative code into clear, declarative expressions
   - Make common patterns more readable and maintainable
   - Hide implementation complexity behind intuitive interfaces

2. **Ensure Safety**
   - Leverage Kotlin's type system to prevent errors at compile time
   - Use @DslMarker to prevent scope ambiguity
   - Make invalid states and unsafe operations impossible

3. **Scale with Complexity**
   - Start simple with basic builders like buildString
   - Handle complex scenarios like LazyColumn's recycling
   - Maintain clarity even as functionality grows

The success of DSLs in Kotlin shows their versatility across different domains:
- Basic utilities (buildString)
- UI frameworks (Compose)
- Configuration systems
- Any API where clarity and safety matter

By understanding these patterns and tools, from extension functions to @DslMarker, you can create APIs that are not only powerful and safe but also a pleasure to use.
