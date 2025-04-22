---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Leveraging expect/actual in Kotlin Multiplatform for Native Implementations"
date: 2025-04-22T08:00:00+01:00
description: "How to use Kotlin Multiplatform's expect/actual mechanism to create platform-specific implementations with shared interfaces."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/expect-actual-post.png
draft: false
tags: 
- kotlin
- multiplatform
- kmp
---

### Leveraging expect/actual in Kotlin Multiplatform for Native Implementations

Kotlin Multiplatform (KMP) has emerged as a powerful solution for sharing code across different platforms while still allowing for platform-specific implementations when needed. At the heart of this capability is the expect/actual mechanism, which enables developers to define a common API in shared code and provide platform-specific implementations. This blog post explores how to effectively use expect/actual to create robust multiplatform applications with native implementations.

---

#### Understanding expect/actual in Kotlin Multiplatform

The expect/actual mechanism is Kotlin's approach to handling platform-specific code in a multiplatform project. It consists of two key components:

1. **expect declarations**: Define what functionality is required in the common code
2. **actual implementations**: Provide platform-specific implementations of that functionality

```kotlin
// In common code
expect class PlatformDateFormatter() {
    fun format(date: Long): String
}

// In Android-specific code
actual class PlatformDateFormatter {
    actual fun format(date: Long): String {
        val dateFormat = SimpleDateFormat("yyyy-MM-dd HH:mm:ss", Locale.getDefault())
        return dateFormat.format(Date(date))
    }
}

// In iOS-specific code
actual class PlatformDateFormatter {
    actual fun format(date: Long): String {
        val dateFormatter = NSDateFormatter()
        dateFormatter.dateFormat = "yyyy-MM-dd HH:mm:ss"
        return dateFormatter.stringFromDate(NSDate(timeIntervalSince1970 = date / 1000.0))
    }
}
```

The expect declaration serves as a contract that must be fulfilled by each platform-specific implementation. This ensures that the common code can rely on certain functionality being available, regardless of the platform.

---

#### Advantages of Using expect/actual in KMP

The expect/actual mechanism offers several significant benefits for multiplatform development:

1. **Code Sharing with Platform-Specific Optimizations**
   - Share business logic, models, and algorithms across platforms
   - Implement platform-specific optimizations where needed
   - Leverage platform-native APIs for better performance and user experience

2. **Type Safety Across Platforms**
   - The compiler ensures that all expected declarations have corresponding actual implementations
   - Type checking works across platform boundaries
   - Refactoring is safer as changes to expect declarations must be reflected in all actual implementations

3. **Better Developer Experience**
   - Clear separation between shared interfaces and platform-specific implementations
   - IDE support for navigating between expect and actual declarations
   - Easier maintenance as the common API is defined in one place

4. **Gradual Adoption Path**
   - Start with platform-specific code and gradually move to shared implementations
   - Selectively choose which components to share and which to keep platform-specific
   - Integrate with existing codebases without complete rewrites

---

#### Practical Examples of expect/actual in Action

Let's explore some practical examples of how expect/actual can be used in real-world applications.

##### Example 1: Platform-Specific Storage

```kotlin
// In commonMain
expect class LocalStorage {
    fun saveString(key: String, value: String)
    fun getString(key: String): String?
    fun clear()
}

// In androidMain
actual class LocalStorage {
    private val sharedPreferences = context.getSharedPreferences("app_prefs", Context.MODE_PRIVATE)

    actual fun saveString(key: String, value: String) {
        sharedPreferences.edit().putString(key, value).apply()
    }

    actual fun getString(key: String): String? {
        return sharedPreferences.getString(key, null)
    }

    actual fun clear() {
        sharedPreferences.edit().clear().apply()
    }
}

// In iosMain
actual class LocalStorage {
    private val userDefaults = NSUserDefaults.standardUserDefaults

    actual fun saveString(key: String, value: String) {
        userDefaults.setObject(value, key)
    }

    actual fun getString(key: String): String? {
        return userDefaults.stringForKey(key)
    }

    actual fun clear() {
        userDefaults.dictionaryRepresentation().keys.forEach {
            userDefaults.removeObjectForKey(it)
        }
    }
}
```

This example demonstrates how to create a common storage interface while leveraging platform-specific storage mechanisms (SharedPreferences on Android and NSUserDefaults on iOS).

---

##### Example 2: Network Connectivity Monitoring

```kotlin
// In commonMain
expect class NetworkMonitor() {
    fun startMonitoring(onConnectivityChange: (Boolean) -> Unit)
    fun stopMonitoring()
}

// In androidMain
actual class NetworkMonitor {
    private var connectivityManager: ConnectivityManager? = null
    private var networkCallback: ConnectivityManager.NetworkCallback? = null

    actual fun startMonitoring(onConnectivityChange: (Boolean) -> Unit) {
        connectivityManager = context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
        networkCallback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) {
                onConnectivityChange(true)
            }

            override fun onLost(network: Network) {
                onConnectivityChange(false)
            }
        }

        val networkRequest = NetworkRequest.Builder().build()
        connectivityManager?.registerNetworkCallback(networkRequest, networkCallback!!)
    }

    actual fun stopMonitoring() {
        networkCallback?.let { callback ->
            connectivityManager?.unregisterNetworkCallback(callback)
        }
    }
}

// In iosMain
actual class NetworkMonitor {
    private var reachability: SCNetworkReachability? = null

    actual fun startMonitoring(onConnectivityChange: (Boolean) -> Unit) {
        reachability = SCNetworkReachabilityCreateWithName(null, "www.apple.com")

        SCNetworkReachabilitySetCallback(reachability) { _, flags, _ ->
            val isReachable = flags.contains(SCNetworkReachabilityFlags.Reachable)
            onConnectivityChange(isReachable)
        }

        SCNetworkReachabilityScheduleWithRunLoop(reachability, CFRunLoopGetMain(), kCFRunLoopCommonModes)
    }

    actual fun stopMonitoring() {
        reachability?.let { reach ->
            SCNetworkReachabilityUnscheduleFromRunLoop(reach, CFRunLoopGetMain(), kCFRunLoopCommonModes)
        }
    }
}
```

This example shows how to monitor network connectivity using platform-specific APIs while maintaining a consistent interface in shared code.

---

#### Best Practices for Using expect/actual

To get the most out of the expect/actual mechanism, consider these best practices:

1. **Keep expect declarations minimal**
   - Define only what's necessary for the common code to function
   - Avoid exposing platform-specific details in the expect declaration
   - Use interfaces when possible to define behavior rather than implementation

2. **Use expect/actual strategically**
   - Not everything needs to be an expect/actual declaration
   - Consider alternatives like interface implementations for simpler cases
   - Reserve expect/actual for cases where you need deep platform integration

3. **Organize your code effectively**
   - Follow the standard KMP source set structure (commonMain, androidMain, iosMain, etc.)
   - Group related expect/actual declarations together
   - Consider using separate files for complex expect/actual implementations

4. **Handle platform-specific features gracefully**
   - Use expect/actual to provide fallbacks for features not available on all platforms
   - Consider optional functionality that degrades gracefully
   - Document platform-specific limitations clearly

5. **Test both common and platform-specific code**
   - Write tests for the common interface in commonTest
   - Create platform-specific tests for actual implementations
   - Use mocks or test doubles when appropriate

---

#### Advanced Patterns with expect/actual

As you become more comfortable with expect/actual, you can leverage more advanced patterns:

##### Delegating to Platform-Specific Libraries

```kotlin
// In commonMain
expect class JsonParser() {
    fun parse(jsonString: String): Map<String, Any?>
    fun stringify(map: Map<String, Any?>): String
}

// In androidMain
actual class JsonParser {
    private val gson = Gson()

    actual fun parse(jsonString: String): Map<String, Any?> {
        val type = object : TypeToken<Map<String, Any?>>() {}.type
        return gson.fromJson(jsonString, type)
    }

    actual fun stringify(map: Map<String, Any?>): String {
        return gson.toJson(map)
    }
}

// In iosMain
actual class JsonParser {
    actual fun parse(jsonString: String): Map<String, Any?> {
        val nsString = NSString.create(string = jsonString)
        val data = nsString.dataUsingEncoding(NSUTF8StringEncoding)
        val nsObject = NSJSONSerialization.JSONObjectWithData(data!!, 0, null)
        return nsObject as Map<String, Any?>
    }

    actual fun stringify(map: Map<String, Any?>): String {
        val nsData = NSJSONSerialization.dataWithJSONObject(map, 0, null)
        return NSString.create(data = nsData!!, encoding = NSUTF8StringEncoding) as String
    }
}
```

This pattern allows you to leverage platform-specific libraries (Gson for Android and NSJSONSerialization for iOS) while maintaining a consistent API.

---

### Conclusion

The expect/actual mechanism is a cornerstone of Kotlin Multiplatform development, enabling developers to write shared code while still leveraging platform-specific capabilities. By defining a common interface with expect declarations and providing platform-specific implementations with actual declarations, you can create applications that share business logic while taking advantage of native platform features.

As you build multiplatform applications, remember that expect/actual is just one tool in your KMP toolkit. Use it judiciously, alongside other approaches like interfaces and abstract classes, to create the right balance of code sharing and platform-specific optimization.

With the right approach to expect/actual, you can significantly reduce code duplication, improve maintainability, and deliver high-quality applications across multiple platforms without sacrificing the native experience that users expect.
