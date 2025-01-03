---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Using Koin in Compose Multiplatform"
date: 2025-01-02T08:00:00+01:00
description: "Using Koin in Compose Multiplatform from common code with the possibility of configuring each platform."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/koin-cmp.png
draft: false
tags: 
- kotlin
- multiplatform
- cmp
- compose
- koin
---

# Using Koin in Compose Multiplatform

Dependency injection is a must-have for scalable applications, and Koin makes it straightforward, even in Compose Multiplatform projects. With the new `KoinApplication` composable function, you can initialize Koin directly from `commonMain` code, reducing boilerplate while maintaining platform-specific flexibility. Let’s walk through an example.

---

## Project Setup

Start by creating a Compose Multiplatform project using the [KMP Wizard](https://kmp.jetbrains.com/), selecting Android, iOS, Desktop, and Web targets. For this example, we won’t include a server target.

---

## Adding Dependencies

Use Gradle’s version catalog to include the necessary Koin dependencies in `libs.versions.toml`:

```toml
[versions]
koin-bom = "4.1.0-Beta1"

[libraries]
koin-bom = { module = "io.insert-koin:koin-bom", version.ref = "koin-bom" }
koin-core = { module = "io.insert-koin:koin-core" }
koin-android = { module = "io.insert-koin:koin-android" }
koin-compose = { module = "io.insert-koin:koin-compose" }
koin-compose-viewModel = { module = "io.insert-koin:koin-compose-viewmodel" }
```

---

## Defining Koin Modules

We’ll create two Koin modules: `appModule` and `platformModule`. The `platformModule` defines platform-specific dependencies.

### Shared Code Modules

```kotlin
val appModule = module {
    viewModelOf(::MainViewModel)
    factoryOf(::GetJokeUseCase)
    singleOf(::DefaultJokeRepository) bind JokeRepository::class
    singleOf(::JokeJsonDataSource) bind JokeDataSource::class
    single { Json { ignoreUnknownKeys = true } }
}

val Module.localPreferencesDefinition
    get() = singleOf(::InMemoryLocalPreferences) bind LocalPreferences::class

expect val platformModule: Module
```

### Platform-Specific Modules

For Android we will use an implementation to the LocalPreferences interface that depends on Android Context so we need a different module declaration:

```kotlin
actual val platformModule: Module
    get() = module {
        singleOf(::AndroidPreferences) bind LocalPreferences::class
    }
```

For iOS, Desktop, and Web, we reuse the `localPreferencesDefinition`:

```kotlin
actual val platformModule: Module
    get() = module {
        localPreferencesDefinition
    }
```

---

## Configuring the App

In `App.kt`, we’ll use the `KoinApplication` composable function to initialize Koin. Adding `KoinAppDeclaration` as optional parameter and default to null.

```kotlin
@Composable
@Preview
fun App(koinAppDeclaration: KoinAppDeclaration? = null) {
    KoinApplication(
        application = {
            koinAppDeclaration?.invoke(this)
            modules(appModule, platformModule)
        }
    ) {
        MaterialTheme {
            MainScreen()
        }
    }
}
```

On Android, we pass a lambda to provide the context and enable logging:

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            App {
                androidLogger(Level.DEBUG)
                androidContext(this@MainActivity)
            }
        }
    }
}
```

This flexibility ensures platform-specific setups, like injecting the Android context, without affecting other targets.

---

## Run the app

Run the app on each platform. You can test that all is working fine and each platform gets the configuration needed to run.

## Conclusion

The new `KoinApplication` composable simplifies dependency injection in Compose Multiplatform by allowing shared initialization with platform-specific tweaks. This approach reduces boilerplate and enhances code reusability across targets.

Check out the complete example on [GitHub](https://github.com/IgnacioCarrionN/KMPKoin).

Also if you need more info about the different options to declare dependencies on *Koin* you can refer to a post I made on LinkedIn: [Koin DSL](https://www.linkedin.com/posts/nacho-carrion_koin-declaring-dependencies-activity-7279802828687101953-W3rs?utm_source=share&utm_medium=member_desktop) 