---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Testing in Compose Multiplatform (CMP) from Common Code"
date: 2025-02-03T08:00:00+01:00
description: "Learn how to write and run UI tests in Compose Multiplatform (CMP) from common code, ensuring cross-platform consistency for Android, iOS, and Desktop."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/compose-test.png
draft: false
tags:
- kotlin
- compose
- cmp
- multiplatform
---

# **Testing in Compose Multiplatform (CMP) from Common Code**

Compose Multiplatform (CMP) enables building UI for multiple platforms using Jetpack Compose. Fortunately, CMP also supports **writing and running UI tests in the common code**, making testing more efficient across platforms. In this post, we’ll explore how to test CMP applications using `compose.uiTest` and run them on Android, Desktop, and iOS.

---  

## **1. Setting Up Common UI Testing**

CMP provides `compose.uiTest`, allowing UI tests to be written in the shared module without platform-specific dependencies. This means you can **write once and test everywhere**.

### **Updating Build Configuration**

To enable testing, update your `build.gradle.kts` file in the shared module:

```kotlin
kotlin {
    androidTarget {
        instrumentedTestVariant.sourceSetTree.set(KotlinSourceSetTree.test)
    }

    sourceSets {
        val commonTest by getting {
            dependencies {
                implementation(kotlin("test"))
                @OptIn(org.jetbrains.compose.ExperimentalComposeLibrary::class)
                implementation(compose.uiTest) // Common UI testing library
            }
        }
        val desktopTest by getting {
            dependencies {
                implementation(compose.desktop.currentOs)
            }
        }
    }
}
```

### **Declaring Android UI Test Dependencies**

At the root level of your `build.gradle.kts` file, add the necessary Android test dependencies:

```kotlin
dependencies {
    androidTestImplementation("androidx.compose.ui:ui-test-junit4-android:1.5.4")
    debugImplementation("androidx.compose.ui:ui-test-manifest:1.5.4")
}
```

### **Setting Up Android-Specific Test Configuration**

For **Android instrumentation tests**, add the following to your `build.gradle.kts`:

```kotlin
android {
    defaultConfig {
        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
    }
}
```

---  

## **2. Implementing a Simple Counter UI**

Let’s create a **CounterViewModel** that will be tested:

```kotlin
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow

class CounterViewModel {
    private val _count = MutableStateFlow(0)
    val count: StateFlow<Int> = _count.asStateFlow()

    fun increment() {
        _count.value += 1
    }
}
```

Now, let’s create the **Composable UI** that interacts with this ViewModel:

```kotlin
import androidx.compose.foundation.layout.*
import androidx.compose.material.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.testTag
import androidx.compose.ui.unit.dp

@Composable
fun CounterScreen(viewModel: CounterViewModel) {
    val count by viewModel.count.collectAsState()

    Column(
        modifier = Modifier.padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(
            text = count.toString(), 
            fontSize = 24.sp, 
            modifier = Modifier.testTag("counterText")
        )
        Spacer(modifier = Modifier.height(8.dp))
        Button(
            onClick = { viewModel.increment() },
            modifier = Modifier.testTag("incrementButton")
        ) {
            Text("Increment")
        }
    }
}
```

---  

## **3. Writing a Common UI Test in CMP**

Now, let’s write a **UI test** in `commonTest` to validate that clicking the button increments the counter:

```kotlin
import androidx.compose.ui.test.*
import kotlin.test.Test
import kotlin.test.assertEquals

class CounterScreenTest {

    @OptIn(ExperimentalTestApi::class)
    @Test
    fun testButtonIncrementsCounter() = runComposeUiTest {
        val viewModel = CounterViewModel()

        setContent {
            CounterScreen(viewModel = viewModel)
        }

        onNodeWithTag("counterText").assertTextEquals("0")
        onNodeWithTag("incrementButton").performClick()
        onNodeWithTag("counterText").assertTextEquals("1")
    }
}
```

---  

## **4. Running the Tests on Multiple Platforms**

Now that we have our test written in common code, let’s **execute it on Android, Desktop, and iOS**.

### **Run Tests on Android**

For **Android instrumented tests**, run:

```bash
./gradlew connectedAndroidTest
```

### **Run Tests on Desktop**

```bash
./gradlew desktopTest
```

### **Run Tests on iOS**

For **iOS simulator tests**, run:

```bash
./gradlew :composeApp:iosSimulatorArm64Test
```

---  

## **5. Why Test CMP from Common Code?**

✅ **Write once, test everywhere:** No need to duplicate tests across platforms.  
✅ **Consistent behavior across platforms:** Ensures UI elements function the same way.  
✅ **Easier maintenance:** Single test suite covering all targets.

---  

## **Conclusion**

With **Compose Multiplatform UI testing**, we can validate UI behavior **from shared code** without requiring platform-specific test implementations. The `compose.uiTest` library allows us to test UI interactions like text verification and button clicks, ensuring UI consistency across Android, iOS, and Desktop.
