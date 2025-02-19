---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Reliable Timekeeping with the TrustedTime API in Android"
date: 2025-02-19T08:00:00+01:00
description: "The TrustedTime API leverages Google's secure infrastructure to offer a trustworthy timestamp"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/trusted-time-api.png
draft: false
tags:
- kotlin
- android
- google
---

# Reliable Timekeeping with the TrustedTime API in Android

Accurate timekeeping is crucial for many app functionalities, including scheduling, transaction logging, and security. However, relying on a device's system clock can be problematic since users can alter their device’s time settings. To address this, Google has introduced the TrustedTime API, providing a reliable and tamper-resistant time source for Android apps.

## Understanding the TrustedTime API

The TrustedTime API leverages Google's secure infrastructure to offer a trustworthy timestamp, independent of the device's local time settings. It periodically syncs with Google's accurate time servers, reducing the need for frequent network requests. The API also accounts for clock drift, alerting developers when time accuracy may degrade between synchronizations.

## Why Reliable Timekeeping Matters

Relying solely on a device’s clock can cause issues such as:

- **Data Inconsistency:** Apps that depend on event ordering can face data corruption if users manipulate device time.
- **Security Risks:** Time-based security measures, like OTPs and access controls, require an accurate clock.
- **Unreliable Scheduling:** Reminders and scheduled events may malfunction if the device clock is incorrect.
- **Clock Drift:** Internal clocks can drift due to factors like temperature changes and battery levels.
- **Multi-Device Synchronization Issues:** Inconsistent time settings across devices can disrupt data sync.
- **Excessive Battery & Data Usage:** Constantly querying network time servers drains resources, which TrustedTime helps optimize.

## Practical Applications of TrustedTime

The TrustedTime API enhances security and reliability in various scenarios:

- **Financial Apps:** Ensures accurate timestamps for transactions.
- **Gaming:** Prevents time-based exploits.
- **E-Commerce:** Tracks order processing and delivery accurately.
- **Limited-Time Offers:** Ensures promotions expire correctly.
- **IoT Devices:** Synchronizes clocks across multiple devices.
- **Productivity Apps:** Maintains accurate timestamps for cloud document edits.

## Integrating TrustedTime into Your Android App

### 1. Add the Dependency

Include the TrustedTime API in your `build.gradle` file:

```groovy
implementation 'com.google.android.gms:play-services-time:16.0.1'
```

### 2. Initialize TrustedTimeClient

```kotlin
class MyApp : Application() {
    var trustedTimeClient: TrustedTimeClient? = null
        private set

    override fun onCreate() {
        super.onCreate()
        TrustedTime.createClient(this).addOnCompleteListener { task ->
            if (task.isSuccessful) {
                trustedTimeClient = task.result
            } else {
                // Handle error
            }
        }
    }
}
```

### 3. Using TrustedTime in an Activity or Fragment

To retrieve the trusted time within an Activity or Fragment, we can access the `TrustedTimeClient` from the application class:

```kotlin
class MainActivity : AppCompatActivity() {
    private val trustedTimeClient: TrustedTimeClient?
        get() = (application as MyApp).trustedTimeClient

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        getTrustedTime()
    }

    private fun getTrustedTime() {
        val currentTimeMillis = trustedTimeClient?.computeCurrentUnixEpochMillis()
            ?: System.currentTimeMillis()
        Log.d("TrustedTime", "Trusted current time: $currentTimeMillis")
    }
}
```

By retrieving `trustedTimeClient` from `MyApp`, we avoid redundant initialization and ensure a single trusted source of time throughout the app.

## Considerations & Limitations

- **Requires Internet Access:** TrustedTime needs an internet connection after boot to function correctly.
- **Clock Drift:** While the API estimates error, it cannot entirely prevent clock drift.
- **Tampering Protection:** It reduces, but does not completely eliminate, time manipulation risks.
- **TrustedTime API Availability and Limitations:** The TrustedTime API is available on all devices running Google Play services on Android 5 (Lollipop) and above.

By integrating the TrustedTime API, developers can improve the accuracy and security of time-dependent app functionalities, ensuring a consistent and reliable experience for users.
