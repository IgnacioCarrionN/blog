---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Implementing Navigation in Compose Multiplatform Projects"
date: 2025-05-16T08:00:00+01:00
description: "A comprehensive guide to implementing org.jetbrains.androidx.navigation in Compose Multiplatform projects, with a focus on the stable iOS support in Compose Multiplatform 1.8.0."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/compose-navigation-kmp.png
draft: false
tags: 
- kotlin
- multiplatform
- kmp
- compose
- navigation
---

### Implementing Navigation in Compose Multiplatform Projects

With the latest release of Compose Multiplatform (1.8.0), iOS support has been declared stable, marking a significant milestone for cross-platform UI development with Kotlin. One of the key components for building robust applications is navigation, and the org.jetbrains.androidx.navigation library provides a powerful solution that can be integrated into Compose Multiplatform projects. This blog post explores how to implement navigation in a Compose Multiplatform environment, allowing you to share navigation logic across Android and iOS platforms.

---

#### Understanding Navigation in Compose Multiplatform

Navigation in Compose Multiplatform follows the same principles as navigation in Jetpack Compose for Android, but with adaptations to work across multiple platforms. The navigation architecture consists of:

1. **NavController**: The central API for navigation that keeps track of back stack entries and handles navigation operations
2. **NavHost**: A composable that displays the current destination of a NavController
3. **NavGraph**: A collection of destinations that defines possible navigation paths

In a Compose Multiplatform context, navigation:

1. Allows for shared navigation logic across platforms
2. Supports platform-specific navigation requirements when needed

This approach enables us to define our navigation structure in common code, while still accommodating platform-specific behaviors when necessary.

```kotlin
// In commonMain - Basic navigation setup
@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(navController = navController, startDestination = "home") {
        composable("home") {
            HomeScreen(
                onNavigateToDetails = { itemId ->
                    navController.navigate("details/$itemId")
                }
            )
        }
        composable(
            route = "details/{itemId}",
            arguments = listOf(navArgument("itemId") { type = NavType.StringType })
        ) { backStackEntry ->
            val itemId = backStackEntry.arguments?.getString("itemId") ?: ""
            DetailsScreen(
                itemId = itemId,
                onNavigateBack = {
                    navController.popBackStack()
                }
            )
        }
    }
}
```

---

#### Setting Up Navigation in a Compose Multiplatform Project

To integrate navigation into your Compose Multiplatform project, follow these steps:

##### 1. Configure the build.gradle.kts file in your shared module

```kotlin
plugins {
    kotlin("multiplatform")
    id("com.android.library")
    id("org.jetbrains.compose")
}

kotlin {
    androidTarget()
    iosX64()
    iosArm64()
    iosSimulatorArm64()

    sourceSets {
        val commonMain by getting {
            dependencies {
                // Compose Multiplatform
                implementation(compose.runtime)
                implementation(compose.foundation)
                implementation(compose.material)

                // Navigation for Compose
                implementation("org.jetbrains.androidx.navigation:navigation-compose:2.8.0")
            }
        }
    }
}
```

##### 2. Create a shared NavHost in common code

```kotlin
// In commonMain/kotlin/navigation/AppNavigation.kt

@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(
        navController = navController,
        startDestination = "home"
    ) {
        // Define your navigation graph here
        composable("home") {
            HomeScreen(
                onNavigateToProfile = {
                    navController.navigate("profile")
                },
                onNavigateToSettings = {
                    navController.navigate("settings")
                }
            )
        }

        composable("profile") {
            ProfileScreen(
                onNavigateBack = {
                    navController.popBackStack()
                }
            )
        }

        composable("settings") {
            SettingsScreen(
                onNavigateBack = {
                    navController.popBackStack()
                }
            )
        }
    }
}
```

---

#### Platform-Specific Considerations

##### Android Implementation

On Android, navigation integrates seamlessly with the platform's lifecycle:

```kotlin
// In androidMain/kotlin/MainActivity.kt

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        setContent {
            AppTheme {
                AppNavigation()
            }
        }
    }
}
```

##### iOS Implementation

With Compose Multiplatform 1.8.0, iOS support is now stable, making navigation implementation more reliable:

```kotlin
// In iosMain/kotlin/MainViewController.kt

fun MainViewController() = ComposeUIViewController {
    AppTheme {
        AppNavigation()
    }
}
```

---

#### Advanced Navigation Techniques

##### 1. Nested Navigation Graphs

For more complex applications, you can organize your navigation with nested graphs:

```kotlin
@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(navController = navController, startDestination = "home") {
        composable("home") { HomeScreen(navController) }

        // Nested navigation graph for authentication flow
        navigation(startDestination = "login", route = "auth") {
            composable("login") { LoginScreen(navController) }
            composable("register") { RegisterScreen(navController) }
            composable("forgot_password") { ForgotPasswordScreen(navController) }
        }

        // Nested navigation graph for settings
        navigation(startDestination = "settings_main", route = "settings") {
            composable("settings_main") { SettingsMainScreen(navController) }
            composable("appearance") { AppearanceSettingsScreen(navController) }
            composable("notifications") { NotificationSettingsScreen(navController) }
        }
    }
}
```

##### 2. Passing Arguments Between Destinations

```kotlin
// Define routes with arguments
private object Routes {
    const val ITEM_DETAILS = "item_details/{itemId}"

    fun itemDetails(itemId: String) = "item_details/$itemId"
}

// In your NavHost
NavHost(navController = navController, startDestination = "items_list") {
    composable("items_list") {
        ItemsListScreen(
            onItemClick = { itemId ->
                navController.navigate(Routes.itemDetails(itemId))
            }
        )
    }

    composable(
        route = Routes.ITEM_DETAILS,
        arguments = listOf(navArgument("itemId") { type = NavType.StringType })
    ) { backStackEntry ->
        val itemId = backStackEntry.arguments?.getString("itemId") ?: ""
        ItemDetailsScreen(itemId = itemId)
    }
}
```

##### 3. Handling Deep Links

```kotlin
NavHost(
    navController = navController,
    startDestination = "home"
) {
    composable(
        route = "home",
        deepLinks = listOf(
            navDeepLink { uriPattern = "https://example.com/home" }
        )
    ) {
        HomeScreen()
    }

    composable(
        route = "product/{productId}",
        arguments = listOf(navArgument("productId") { type = NavType.StringType }),
        deepLinks = listOf(
            navDeepLink { 
                uriPattern = "https://example.com/product/{productId}" 
            }
        )
    ) { backStackEntry ->
        val productId = backStackEntry.arguments?.getString("productId") ?: ""
        ProductScreen(productId = productId)
    }
}
```

##### 4. Type-Safety with Kotlin DSL (Navigation 2.8.0+)

Starting from Navigation 2.8.0, you can use a type-safe Kotlin DSL to define your navigation graph using @Serializable annotations and data classes, which provides compile-time safety for routes and arguments without needing to define routes as strings:

```kotlin
// Define your destinations with type-safe arguments
@Serializable
object Home

// Define a profile route that takes an ID
@Serializable
data class Profile(val id: String)

// Create a type-safe navigation graph
@Composable
fun TypeSafeNavigation() {
    val navController = rememberNavController()

    NavHost(navController = navController, startDestination = Home) {
        composable<Home> {
            HomeScreen(onNavigateToProfile = { id ->
                navController.navigate(Profile(id))
            })
        }

        composable<Profile> { backStackEntry ->
            val profile: Profile = backStackEntry.toRoute()
            ProfileScreen(profile.id)
        }
    }
}

// Navigate with type-safety
navController.navigate(Profile(id = "123"))
```

Benefits of using the type-safe DSL with @Serializable:

1. **No string routes**: No need to define route strings or string templates
2. **Fully type-safe**: Arguments are part of the route data structure
3. **Compile-time safety**: Typos and type mismatches are caught at compile time
4. **Direct object access**: Access route parameters directly as properties of the route object
5. **Simplified navigation**: Navigate by passing objects directly to the navigate function

To enable the type-safe DSL, make sure you're using Navigation 2.8.0 or higher:

```kotlin
implementation("org.jetbrains.androidx.navigation:navigation-compose:2.8.0")
```

---

#### Best Practices for Navigation in Compose Multiplatform

1. **Create a navigation abstraction layer**
   - Define a common interface for navigation actions
   - Implement platform-specific details behind this interface
   - This makes it easier to handle platform differences

2. **Use type-safe navigation with @Serializable**
   - Leverage the type-safe DSL with @Serializable annotations instead of string routes
   - Define destinations as data classes and objects for compile-time safety
   - Access route parameters directly as properties of route objects
   - This eliminates string-based routes, prevents typos, and makes refactoring easier

3. **Manage navigation state properly**
   - Consider using a state management solution like ViewModel
   - Keep navigation logic separate from UI logic
   - This makes your code more testable and maintainable

4. **Test your navigation flows**
   - Write tests for your navigation logic
   - Verify that arguments are passed correctly
   - Ensure deep links work as expected

---

### Conclusion

With Compose Multiplatform 1.8.0 declaring iOS support as stable, implementing navigation in cross-platform applications has become more reliable and straightforward. The approach outlined in this post provides a practical way to share navigation logic across platforms while still accommodating platform-specific requirements.

By following the configuration steps, platform-specific considerations, and best practices outlined in this post, you can successfully implement navigation in your Compose Multiplatform projects and create consistent, intuitive user experiences across Android and iOS platforms.

For more detailed information, refer to the [official Compose Multiplatform navigation documentation](https://www.jetbrains.com/help/kotlin-multiplatform-dev/compose-navigation.html).
