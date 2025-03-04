---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Achieving Compile-Time Safety in Koin: A Comprehensive Guide"
date: 2025-03-04T08:00:00+01:00
description: "Dependency injection is a fundamental pattern in modern Android development, but how can we ensure our DI configuration is correct before running the app? In this post, we'll explore two powerful approaches to achieve compile-time safety with Koin: using the DSL's `verify()` function and leveraging Koin Annotations with KSP."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/koin-ksp-config.png
draft: false
tags:
- kotlin
- android
- koin
---

# Achieving Compile-Time Safety in Koin: A Comprehensive Guide

Dependency injection is a fundamental pattern in modern Android development, but how can we ensure our DI configuration is correct before running the app? In this post, we'll explore two powerful approaches to achieve compile-time safety with Koin: using the DSL's `verify()` function and leveraging Koin Annotations with KSP.

## The Problem: Runtime vs. Compile-Time Validation

Traditional dependency injection often reveals configuration issues only at runtime:
- Missing dependencies cause crashes
- Circular dependencies aren't detected until runtime
- Type mismatches surface when the app is running
- Wrong scoping leads to unexpected behavior

These issues can be particularly problematic in large applications where dependency graphs are complex and not all paths are covered by tests.

## Solution 1: Koin DSL with verify()

The first approach uses Koin's built-in `verify()` function to check module configurations during the compilation phase.

### How It Works

```kotlin
val appModule = module {
    single<Repository> { DefaultRepository() }
    factory { UseCase(get()) }
    viewModel { MainViewModel(get()) }
}

class ModuleCheck {
    @Test
    fun verifyKoinModules() {
        appModule.verify()
    }
}
```

By creating a unit test that calls `verify()` on your modules and making it part of your build process, you can catch common issues early.

### Advantages
- Simple to implement
- No additional dependencies required
- Works with existing Koin DSL code
- Can be integrated into CI/CD pipelines

### Limitations
- Requires explicit test execution
- Less IDE support
- No direct compilation errors
- Can't catch all potential issues

## Solution 2: Koin Annotations with KSP

The second approach uses Kotlin Symbol Processing (KSP) and Koin's annotation system to validate dependency graphs during compilation.

### Setup

```kotlin
// build.gradle.kts
plugins {
    id("com.google.devtools.ksp") version "2.1.10-1.0.31"
}

dependencies {
    implementation("io.insert-koin:koin-annotations:2.0.0-RC5")
    ksp("io.insert-koin:koin-ksp-compiler:2.0.0-RC5")
}

ksp {
    arg("KOIN_CONFIG_CHECK", "true")
}
```

### How It Works

Instead of using the DSL, you define dependencies using annotations:

```kotlin
@SingleOf(binds = [Repository::class])
class DefaultRepository : Repository {
    // Implementation
}

@Module
class AppModule {
    @Factory
    fun provideUseCase(repository: Repository): UseCase {
        return UseCase(repository)
    }
}

@Module
class ViewModelModule {
    @KoinViewModel
    fun provideMainViewModel(
        useCase: UseCase
    ): MainViewModel = MainViewModel(useCase)
}
```

KSP processes these annotations during compilation and:
- Generates type-safe dependency definitions
- Validates the dependency graph
- Ensures all dependencies can be resolved
- Checks for circular dependencies
- Verifies scope consistency

### Advantages
- True compile-time validation
- Better IDE support
- More explicit dependency declarations
- Catches issues during compilation
- Type-safe by design

### Limitations
- Requires additional setup
- More verbose than DSL
- Requires migration from DSL-based code

## Comparing the Approaches

| Feature | DSL + verify() | Annotations + KSP |
|---------|---------------|------------------|
| Setup Complexity | Low | Medium |
| Validation Timing | Build-time (via tests) | Compile-time |
| IDE Support | Limited | Good |
| Migration Effort | Low | Medium |
| Type Safety | Good | Excellent |
| Error Messages | Test failures | Compilation errors |
| Learning Curve | Gentle | Moderate |

## Which Approach Should You Choose?

### Choose DSL + verify() if:
- You have an existing Koin DSL codebase
- You want a simple solution
- You prefer more flexible configuration
- You're comfortable with test-based validation

### Choose Annotations + KSP if:
- You're starting a new project
- You want true compile-time safety
- You prefer explicit dependency declarations
- You value IDE support and type safety

## Best Practices

Regardless of the approach you choose:

1. **Make validation part of your build process**
    - For DSL: Include verification tests in your build
    - For Annotations: Enable KSP validation

2. **Document your dependency graph**
    - Keep a clear structure of modules
    - Document scope decisions
    - Maintain a clean architecture

3. **Monitor build times**
    - Both approaches can impact build times
    - KSP processing adds to compilation time
    - Test execution adds to build time

4. **Consider gradual migration**
    - You can mix both approaches
    - Migrate gradually from DSL to annotations
    - Start with new features using your chosen approach

## Conclusion

Both approaches offer valuable ways to achieve compile-time safety in your Koin dependency injection. The DSL with `verify()` provides a simpler, test-based approach, while annotations with KSP offer stronger compile-time guarantees with better tooling support.

Choose the approach that best fits your project's needs, considering factors like existing codebase, team expertise, and desired level of type safety. Remember that both approaches are significantly better than relying solely on runtime validation.

## Resources
- [Koin Documentation](https://insert-koin.io/)
- [Koin Annotations Guide](https://insert-koin.io/docs/reference/koin-annotations/start)
- [KSP Documentation](https://kotlinlang.org/docs/ksp-overview.html)
- [Sample Project](https://github.com/IgnacioCarrionN/KoinRuntime)
