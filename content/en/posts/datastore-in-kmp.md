---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Implementing DataStore in Kotlin Multiplatform Projects"
date: 2025-05-09T08:00:00+01:00
description: "A comprehensive guide to configuring and using DataStore in Kotlin Multiplatform projects, including platform-specific setup and practical examples."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/datastore-kmp.png
draft: false
tags: 
- kotlin
- multiplatform
- kmp
- datastore
- preferences
---

### Implementing DataStore in Kotlin Multiplatform Projects

DataStore is a modern data storage solution developed by Google as a replacement for SharedPreferences. It provides a consistent, type-safe API for storing key-value pairs and typed objects with Kotlin coroutines and Flow support. With the recent advancements in Kotlin Multiplatform (KMP), we can now integrate DataStore into our KMP projects, allowing us to share preferences and data storage code across platforms. This blog post explores how to configure, implement, and optimize DataStore in a KMP environment.

---

#### Understanding DataStore in Kotlin Multiplatform Context

DataStore in KMP is designed to provide a consistent API across platforms while leveraging platform-specific storage mechanisms. There are two types of DataStore:

1. **Preferences DataStore**: For storing key-value pairs
2. **Proto DataStore**: For storing typed objects using Protocol Buffers

In a KMP context DataStore:

1. Platform-specific implementations provide the actual storage mechanisms
2. The API remains consistent across platforms, using coroutines and Flow

This approach allows us to define our data access patterns in common code, while the underlying storage operations are handled by platform-specific implementations.

```kotlin
// In commonMain - DataStore interface
interface UserPreferences {
    val userData: Flow<UserData>
    suspend fun updateUsername(name: String)
    suspend fun updateEmail(email: String)
    suspend fun clearData()
}

// In commonMain - Data model
data class UserData(
    val username: String = "",
    val email: String = "",
    val isLoggedIn: Boolean = false
)
```

---

#### Setting Up DataStore in a KMP Project

To integrate DataStore into your KMP project, you'll need to configure your build files appropriately. Here's a step-by-step guide:

##### 1. Configure the build.gradle.kts file in your shared module

```kotlin
plugins {
    kotlin("multiplatform")
    id("com.android.library")
    id("com.google.devtools.ksp") version "2.1.20-2.0.1" // For Proto DataStore
}

kotlin {
    androidTarget()
    iosX64()
    iosArm64()
    iosSimulatorArm64()

    sourceSets {
        val commonMain by getting {
            dependencies {
                // For Preferences DataStore
                implementation("androidx.datastore:datastore-preferences-core:1.1.0")

                // For coroutines
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
            }
        }
    }
}
```

##### 2. Create DataStore instance from common code

```kotlin
/**
 * Gets the singleton DataStore instance, creating it if necessary.
 */
fun createDataStore(producePath: () -> String): DataStore<Preferences> =
   PreferenceDataStoreFactory.createWithPath(
      produceFile = { producePath().toPath() }
   )

internal const val dataStoreFileName = "dice.preferences_pb"
```

---

#### Platform-Specific Considerations

##### Android Implementation

```kotlin
// shared/src/androidMain/kotlin/DataStore.kt

fun createDataStoreAndroid(context: Context): DataStore<Preferences> = createDataStore(
   producePath = { context.filesDir.resolve(dataStoreFileName).absolutePath }
)
```

##### iOS Implementation

```kotlin
// shared/src/iosMain/kotlin/DataStore.kt

fun createDataStoreIOS(): DataStore<Preferences> = createDataStore(
   producePath = {
      val documentDirectory: NSURL? = NSFileManager.defaultManager.URLForDirectory(
         directory = NSDocumentDirectory,
         inDomain = NSUserDomainMask,
         appropriateForURL = null,
         create = false,
         error = null,
      )
      requireNotNull(documentDirectory).path + "/$dataStoreFileName"
   }
)
```

---

#### Practical Example: Implementing a User Preferences Repository

To demonstrate a complete implementation, let's create a repository that uses our DataStore:

```kotlin
// In commonMain
class UserPreferencesRepository(private val dataStore: PreferencesDataStore) {
    // Define preference keys
    private object PreferenceKeys {
        val USERNAME = stringPreferencesKey("username")
        val EMAIL = stringPreferencesKey("email")
        val IS_LOGGED_IN = booleanPreferencesKey("is_logged_in")
    }

    // Get user data as a Flow
    val userData: Flow<UserData> = dataStore.data.map { preferences ->
        UserData(
            username = preferences[PreferenceKeys.USERNAME] ?: "",
            email = preferences[PreferenceKeys.EMAIL] ?: "",
            isLoggedIn = preferences[PreferenceKeys.IS_LOGGED_IN] ?: false
        )
    }

    // Update username
    suspend fun updateUsername(name: String) {
        dataStore.updateData { preferences ->
            preferences.toMutablePreferences().apply {
                this[PreferenceKeys.USERNAME] = name
            }
        }
    }

    // Update email
    suspend fun updateEmail(email: String) {
        dataStore.updateData { preferences ->
            preferences.toMutablePreferences().apply {
                this[PreferenceKeys.EMAIL] = email
            }
        }
    }

    // Set login status
    suspend fun setLoggedIn(isLoggedIn: Boolean) {
        dataStore.updateData { preferences ->
            preferences.toMutablePreferences().apply {
                this[PreferenceKeys.IS_LOGGED_IN] = isLoggedIn
            }
        }
    }

    // Clear all data
    suspend fun clearData() {
        dataStore.updateData { preferences ->
            preferences.toMutablePreferences().apply {
                remove(PreferenceKeys.USERNAME)
                remove(PreferenceKeys.EMAIL)
                remove(PreferenceKeys.IS_LOGGED_IN)
            }
        }
    }
}

// In commonMain - ViewModel or Presenter
class UserViewModel(private val userPreferencesRepository: UserPreferencesRepository) {
    val userData: Flow<UserData> = userPreferencesRepository.userData

    suspend fun updateUserProfile(username: String, email: String) {
        if (username.isNotBlank()) {
            userPreferencesRepository.updateUsername(username)
        }

        if (email.isNotBlank()) {
            userPreferencesRepository.updateEmail(email)
        }
    }

    suspend fun login() {
        userPreferencesRepository.setLoggedIn(true)
    }

    suspend fun logout() {
        userPreferencesRepository.setLoggedIn(false)
    }

    suspend fun clearUserData() {
        userPreferencesRepository.clearData()
    }
}
```

---

#### Advanced DataStore Features in KMP

DataStore offers several advanced features that can be leveraged in a KMP environment:

##### 1. Proto DataStore for Typed Objects

If you need to store complex objects, Proto DataStore provides a type-safe solution:

```proto
// Define your data structure in a .proto file
syntax = "proto3";

option java_package = "com.example.app";
option java_multiple_files = true;

message UserPreferences {
  string username = 1;
  string email = 2;
  bool is_logged_in = 3;
}
```

```kotlin
// In commonMain - Create a serializer
class UserPreferencesSerializer : Serializer<UserPreferences> {
    override val defaultValue: UserPreferences = UserPreferences.getDefaultInstance()

    override suspend fun readFrom(input: InputStream): UserPreferences {
        return UserPreferences.parseFrom(input)
    }

    override suspend fun writeTo(t: UserPreferences, output: OutputStream) {
        t.writeTo(output)
    }
}

// Platform-specific implementation for Proto DataStore
```

##### 2. Data Migration

```kotlin
// In androidMain - Migrating from SharedPreferences to DataStore
val dataStore = context.createDataStore(
    name = "user_preferences",
    produceMigrations = { context ->
        listOf(
            SharedPreferencesMigration(
                context = context,
                sharedPreferencesName = "legacy_preferences"
            )
        )
    }
)
```

##### 3. Handling Exceptions

```kotlin
// In commonMain - Handling exceptions during data operations
val userData = dataStore.data
    .catch { exception ->
        // Handle exception (e.g., data corruption)
        if (exception is IOException) {
            emit(emptyPreferences())
        } else {
            throw exception
        }
    }
    .map { preferences ->
        // Map preferences to your data model
        UserData(
            username = preferences[USERNAME] ?: "",
            email = preferences[EMAIL] ?: ""
        )
    }
```

---

#### Best Practices for DataStore in KMP

1. **Leverage coroutines and Flow for asynchronous operations**
   - DataStore operations are asynchronous by design
   - Use Flow to observe changes in your stored data
   - Apply Flow operators like `map`, `filter`, and `combine` for data transformations

2. **Create a repository layer**
   - Abstract DataStore operations behind a repository
   - This makes it easier to switch implementations if needed
   - Provides a clean API for your business logic

3. **Handle errors gracefully**
   - Use `catch` operator to handle exceptions in your Flow
   - Provide fallback values when data cannot be read
   - Consider implementing retry mechanisms for critical operations

4. **Optimize for performance**
   - Minimize the number of DataStore updates
   - Batch related changes together
   - Use `distinctUntilChanged()` to avoid unnecessary emissions

5. **Test your DataStore code thoroughly**
   - Write tests for your repositories in commonTest
   - Use test doubles to simulate different scenarios

---

### Conclusion

Integrating DataStore into a Kotlin Multiplatform project provides a modern, type-safe way to store and access data across platforms.

The approach outlined in this post provides a practical way to share preferences and data storage logic across platforms with minimal platform-specific code. DataStore's support for coroutines and Flow makes it a natural fit for KMP projects, enabling reactive and asynchronous data operations with a consistent API.

By following the configuration steps, platform-specific considerations, and best practices outlined in this post, you can successfully implement DataStore in your KMP projects and create robust, efficient data storage solutions that work across multiple platforms with minimal platform-specific code.
