---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Boosting Android App Performance with Baseline Profiles"
date: 2025-01-27T08:00:00+01:00
description: "Boosting Android App Performance with Baseline Profiles with real world example"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/baseline-profile.png
draft: false
tags:
- kotlin
- architecture
---

# **Boosting Android App Performance with Baseline Profiles**

## **Introduction**

In today’s fast-paced mobile world, users expect apps to launch instantly and run smoothly. Performance optimization is crucial, especially concerning app startup time and runtime execution.

Android’s **Baseline Profiles** offer an effective way to speed up app startup and improve runtime performance by precompiling critical code paths. Google Play even recommends using Baseline Profiles to enhance the user experience, particularly for apps with complex UI rendering or heavy dependencies.

In this guide, we'll explore what Baseline Profiles are, how they work, and how to integrate them into your Android app using a **dedicated Baseline Profile module** to achieve faster startup times and improved runtime performance.

---

## **What Are Baseline Profiles?**

Android apps use the **Android Runtime (ART)** to execute code, employing **Just-In-Time (JIT)** and **Ahead-Of-Time (AOT)** compilation.

However, JIT compilation can introduce delays during app startup since code isn't fully optimized initially. **Baseline Profiles** solve this problem by allowing developers to specify critical code paths that should be precompiled before the app runs.

### **How Do Baseline Profiles Work?**

- **Precompilation**: They contain method and class definitions that are precompiled and optimized before execution.
- **Installation**: These profiles are installed on the user’s device upon the app’s first launch.
- **Optimization**: ART utilizes them to enhance execution speed, especially during cold starts.

---

## **When Should You Use Baseline Profiles?**

Baseline Profiles are particularly beneficial in the following scenarios:

✅ **Optimizing App Startup** – Reducing cold start time by precompiling launch sequences.  
✅ **Improving Scroll Performance** – Ensuring smoother UI rendering.  
✅ **Enhancing Frequently Used Features** – Precompiling logic that users interact with often.

---

## **Setting Up Baseline Profiles in a Separate Module**

### **Step 1: Create the Baseline Profile Module**

1. **Open Android Studio**:
    - Navigate to `File` → `New` → `New Module`.
    - Select the `Baseline Profile Generator` template.

2. **Configure the Module**:
    - **Target Application**: Choose the app module for which the Baseline Profile will be generated.
    - **Module Name**: Assign a name, e.g., `baselineprofile`.
    - **Package Name**: Define the package name for the module.
    - **Language**: Select Kotlin or Java.
    - **Build Configuration Language**: Choose between Kotlin Script (KTS) or Groovy.

3. **Finish**: Click `Finish` to create the module.

This process sets up a new module containing the necessary configurations and code to generate and benchmark Baseline Profiles.

---

### **Step 2: Define the Baseline Profile Generator**

#### **Jetpack Compose-Based Baseline Profile Generator**

```kotlin
@RunWith(AndroidJUnit4::class)
@LargeTest
class BaselineProfileGenerator {

    @get:Rule
    val baselineProfileRule = BaselineProfileRule()

    @get:Rule
    val composeTestRule = createAndroidComposeRule<MainActivity>()

    @Test
    fun baselineProfile() = baselineProfileRule.collect(
        packageName = "com.example.app",
        includeInStartupProfile = true,
        profileBlock = {

            // 1. **App Cold Start**
            startActivityAndWait()

            // 2. **Navigate to Main Screen**
            composeTestRule.onNodeWithText("Get Started")
                .performClick()
            composeTestRule.waitForIdle()

            // 3. **Scroll Through a LazyColumn List**
            composeTestRule.onNodeWithTag("itemList")
                .performScrollToIndex(10)
            composeTestRule.waitForIdle()

            // 4. **Open a Detail Screen**
            composeTestRule.onNodeWithTag("item_0")
                .performClick()
            composeTestRule.waitForIdle()

            // 5. **Perform a Search**
            composeTestRule.onNodeWithTag("searchInput")
                .performTextInput("Kotlin")
            composeTestRule.waitForIdle()

            // 6. **Back Navigation**
            composeTestRule.onNodeWithContentDescription("Back")
                .performClick()
            composeTestRule.waitForIdle()
        }
    )
}
```

---

### **Step 3: Generate and Integrate the Baseline Profile**

1. **Generate the Baseline Profile**:
    - Run the `Generate Baseline Profile` configuration created by the template.
    - Alternatively, execute the following Gradle task:

      ```shell
      ./gradlew :app:generateBaselineProfile
      ```  

   This generates the Baseline Profile and copies it to the appropriate directory in your app module.

2. **Integrate the Baseline Profile**:
    - The generated profile is automatically included in your app module’s assets.
    - Ensure that your app's `build.gradle` includes the necessary dependencies:

      ```kotlin
      dependencies {
          implementation("androidx.profileinstaller:profileinstaller:1.3.0")
          baselineProfile(project(":baselineprofile"))
      }
      ```

   The `profileinstaller` library installs the Baseline Profile on user devices, and the `baselineProfile` dependency links the generated profile to your app.

---

## **Conclusion**

Baseline Profiles are a powerful tool for improving **app startup time and runtime performance** without increasing APK size. By precompiling critical code paths, your app launches faster, providing a **smoother and more responsive experience** for users.

By integrating Baseline Profiles using a **dedicated module**, you ensure a modular, maintainable, and scalable approach to performance optimization.

If you haven’t already, **start using Baseline Profiles today** and measure their impact on your app’s performance!
