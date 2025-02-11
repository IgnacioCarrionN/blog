---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Kotlin Multiplatform Architecture Best Practices for Mobile Apps"
date: 2025-02-11T08:00:00+01:00
description: "Architecture tips for KMP projects using clean architecture"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/expect-actual.png
draft: false
tags:
- kotlin
- compose
- cmp
- multiplatform
- cleancode
- architecture
---

# Kotlin Multiplatform Architecture Best Practices for Mobile Apps

Kotlin Multiplatform (KMP) allows developers to share business logic between Android and iOS while keeping platform-specific implementations where necessary. Structuring a KMP project efficiently is key to maintaining scalability, testability, and clean architecture. In this guide, we’ll explore best practices for architecting a KMP mobile application with **Compose Multiplatform** and **Clean Architecture**.

---

## 1. **Project Structure**

A well-organized project structure improves maintainability and separation of concerns. A common approach is to follow a **multi-module structure**, either with a single shared module or with multiple feature-based modules:

```
project-root/
 ├── core/
 │   ├── network/                  # Network shared logic
 │   │   ├── src/commonMain/       # Shared networking
 │   │   ├── src/androidMain/      # Android-specific implementations
 │   │   ├── src/iosMain/          # iOS-specific implementations
 ├── features/                 # Feature-based modules
 │   ├── feature1/             # Example feature module
 │   │   ├── domain/           # Domain layer (Use cases, repositories interfaces)
 │   │   ├── data/             # Data layer (Implementations, APIs, Database)
 │   │   ├── presentation/     # UI and ViewModels for Compose Multiplatform
 ├── composeApp/               # Application module integrating all features
 │   ├── src/commonMain/       # Can contain shared UI and navigation logic
 │   ├── src/androidMain/      # Android-specific implementations if needed
 │   ├── src/iosMain/          # iOS-specific implementations if needed
 ├── androidApp/               # Android application module
 ├── iosApp/                   # iOS application module
```

- **Feature Modules**: Instead of a single shared module, you can have feature-specific shared modules to improve modularity and scalability. These can be further split into **domain, data, and presentation layers** for better separation of concerns.
- **Core Modules**: Contains shared utilities like networking, logging, and common domain logic.
- **ComposeApp Module**: Acts as the main application module, integrating all feature modules and handling navigation, similar to an `app` module in an Android project.

In most **Compose Multiplatform** projects, a `composeApp` module is used to assemble all features, manage navigation, and handle other app-wide concerns, similar to the `app` module in a standard Android project.

---

## 2. **Applying Clean Architecture in KMP**

Following **Clean Architecture** helps in maintaining separation of concerns and improving testability. The architecture can be structured into the following layers:

### **Domain Layer (commonMain)**

- **Contains business logic** (Use Cases, Interactors).
- **Defines repository interfaces** for data access.
- **Does not depend on any platform-specific implementation.**

```kotlin
interface UserRepository {
    suspend fun getUser(): User
}
```

### **Data Layer (commonMain, platform-specific)**

- **Implements repository interfaces**.
- Uses `expect/actual` for platform-specific APIs like networking, databases, etc.
- Fetches and processes raw data before exposing it to the domain layer.

Example `expect/actual` for HTTP client:

```kotlin
expect class HttpClientProvider {
    fun getClient(): HttpClient
}
```

Android-specific implementation:

```kotlin
actual class HttpClientProvider {
    actual fun getClient() = HttpClient(Android) {}
}
```

iOS-specific implementation:

```kotlin
actual class HttpClientProvider {
    actual fun getClient() = HttpClient(Ios) {}
}
```

### **Presentation Layer (Compose Multiplatform)**

With **Compose Multiplatform**, we can share UI components across platforms while leveraging native rendering. The `composeApp` module integrates all feature modules and handles navigation and app-wide logic.

```kotlin
@Composable
fun UserScreen(viewModel: UserViewModel) {
    val user by viewModel.userState.collectAsState()
    Column(modifier = Modifier.padding(16.dp)) {
        Text(
            "Hello, ${user?.name ?: "Guest"}", 
            style = MaterialTheme.typography.h6
        )
    }
}
```

On **Android**, this is rendered using Jetpack Compose, and on **iOS**, it is rendered using Compose for iOS.

---

## 3. **State Management in KMP**

State management in a KMP project can be handled efficiently using **StateFlow**.

```kotlin
class UserViewModel(private val repository: UserRepository) {
    private val _userState = MutableStateFlow<User?>(null)
    val userState: StateFlow<User?> = _userState

    fun loadUser() {
        viewModelScope.launch {
            _userState.value = repository.getUser()
        }
    }
}
```

Since Compose Multiplatform supports `collectAsState()`, we can observe and render state changes directly in the UI.

---

## 4. **Testing in KMP**

- **Unit Tests in ****`commonTest`** using `kotlin.test`.
- **Platform-specific Tests** in `androidTest` and `iosTest`.

Example shared unit test:

```kotlin
@Test
fun testUserRepository() = runTest {
    val repository = FakeUserRepository()
    assertNotNull(repository.getUser())
}
```

---

## **Conclusion**

By following these best practices, you can build scalable and maintainable KMP applications:

- **Use a modularized project structure** with either a shared module or feature-based modules.
- **Follow Clean Architecture** for maintainability.
- **Leverage Compose Multiplatform** for UI, using a `composeApp` module to integrate feature modules and manage navigation.
- **Feature modules can be further divided into domain, data, and presentation layers** to improve separation of concerns.
- **Manage state efficiently** using `StateFlow`.
- **Write comprehensive tests** across shared and platform-specific code.

KMP enables efficient code sharing while preserving platform-specific optimizations, making it a powerful choice for mobile app development.

Would you like a sample GitHub repository demonstrating this setup? 🚀
