---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Reducing App Size: Proguard, R8, App Bundles & Resource Shrinking"
date: 2025-06-27T08:00:00+01:00
description: "A comprehensive guide to reducing Android app size using Proguard, R8, App Bundles, and Resource Shrinking techniques for better performance and user experience."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/r8.png
draft: false
tags: 
- android
- optimization
- proguard
- r8
- app-bundles
---

### Reducing App Size: Proguard, R8, App Bundles & Resource Shrinking

As mobile applications grow in complexity and feature set, managing app size becomes increasingly important. Large apps lead to higher abandonment rates during installation, consume more device storage, and often result in slower performance. This comprehensive guide explores various techniques to reduce your Android app size, including code shrinking with Proguard and R8, modern delivery with App Bundles, and resource optimization strategies.

---

#### Understanding App Size Components

Before diving into optimization techniques, it's important to understand what contributes to your app's size:

1. **Code**: Java/Kotlin bytecode, including your app code and libraries
2. **Resources**: Images, layouts, animations, and other assets
3. **Native libraries**: C/C++ code compiled for different architectures
4. **Assets**: Raw files bundled with your app

Each component requires different optimization strategies, and a comprehensive approach addresses all of these areas.

```
// A typical app's size breakdown might look like:
// - Code (DEX): 30-40%
// - Resources: 30-40%
// - Native libraries: 15-25%
// - Assets: 5-15%
```

---

#### Code Shrinking with Proguard and R8

Code shrinking is one of the most effective ways to reduce app size by removing unused code and optimizing what remains.

##### Proguard: The Traditional Approach

Proguard has been the standard tool for code shrinking in Android for many years. It performs several optimizations:

1. **Shrinking**: Removes unused classes, fields, methods, and attributes
2. **Optimization**: Optimizes bytecode and removes unused code paths
3. **Obfuscation**: Renames classes, fields, and methods with shorter names
4. **Preverification**: Adds preverification information to simplify the runtime verification process

To enable Proguard in your app's build.gradle file, you need to set minifyEnabled to true and specify the Proguard rules files:

```
android {
    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro"
        }
    }
}
```

A basic Proguard configuration file might include:

```
# Keep the application class
-keep public class com.example.MyApplication

# Keep all classes in the androidx package
-keep class androidx.** { *; }

# Keep all classes that have the @Keep annotation
-keep class * {
    @androidx.annotation.Keep *;
}
-keepclasseswithmembers class * {
    @androidx.annotation.Keep <methods>;
}
-keepclasseswithmembers class * {
    @androidx.annotation.Keep <fields>;
}
```

###### Common Proguard Rules for Popular Libraries

When using third-party libraries, you often need specific Proguard rules to ensure they work correctly after shrinking. Here are examples for some popular libraries:

**Retrofit and OkHttp:**
```
# Retrofit
-keepattributes Signature
-keepattributes *Annotation*
-keep class retrofit2.** { *; }
-keepclasseswithmembers class * {
    @retrofit2.http.* <methods>;
}

# OkHttp
-keepattributes Signature
-keepattributes *Annotation*
-keep class okhttp3.** { *; }
-keep interface okhttp3.** { *; }
-dontwarn okhttp3.**
-dontwarn okio.**
```

**Gson:**
```
# Gson
-keepattributes Signature
-keepattributes *Annotation*
-keep class com.google.gson.** { *; }
-keep class * implements com.google.gson.TypeAdapter
-keep class * implements com.google.gson.TypeAdapterFactory
-keep class * implements com.google.gson.JsonSerializer
-keep class * implements com.google.gson.JsonDeserializer
-keepclassmembers,allowobfuscation class * {
  @com.google.gson.annotations.SerializedName <fields>;
}
```

**Glide:**
```
# Glide
-keep public class * implements com.bumptech.glide.module.GlideModule
-keep class * extends com.bumptech.glide.module.AppGlideModule {
 <init>(...);
}
-keep public enum com.bumptech.glide.load.ImageHeaderParser$** {
  **[] $VALUES;
  public *;
}
```

###### Troubleshooting Common Proguard Issues

When using Proguard, you might encounter some common issues:

1. **ClassNotFoundException or NoSuchMethodError**: This usually happens when a class or method used via reflection is removed or renamed. Solution: Add keep rules for the affected classes.

2. **Missing serialization/deserialization**: JSON parsing libraries like Gson rely on class and field names. Solution: Add rules to preserve names for model classes.

3. **Crashes in third-party libraries**: Some libraries aren't fully compatible with obfuscation. Solution: Check the library's documentation for recommended Proguard rules.

4. **Debugging obfuscated crashes**: Use the mapping.txt file generated during the build to deobfuscate stack traces:

```
// Add this to your build.gradle to keep the mapping file
android {
    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro"
            testProguardFiles getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro"
        }
    }
}
```

##### R8: The Modern Successor

R8 is Android's current default compiler that combines the capabilities of Proguard with additional optimizations:

1. **More aggressive inlining**: Inlines more methods to reduce method count
2. **Class merging**: Combines classes with similar functionality
3. **Enum optimization**: Converts enums to integers when possible
4. **Kotlin-specific optimizations**: Better handling of Kotlin-specific features
5. **Tree shaking**: More aggressive removal of unused code
6. **Desugaring**: Allows using newer Java language features on older Android versions

R8 is enabled by default in Android Gradle Plugin 3.4.0 and higher. You can enable full mode R8 optimization by configuring core library desugaring in your build.gradle file:

```
android {
    compileOptions {
        // Enable core library desugaring
        coreLibraryDesugaringEnabled true
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }

    buildTypes {
        release {
            // Enable R8 in full mode
            minifyEnabled true
            proguardFiles getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro"
        }
    }
}

dependencies {
    // Add the desugaring dependency
    coreLibraryDesugaring "com.android.tools:desugar_jdk_libs:1.1.5"
}
```

###### Comparing Proguard and R8

While R8 is designed to be a drop-in replacement for Proguard, there are some key differences:

| Feature | Proguard | R8 |
|---------|----------|-----|
| Code shrinking | Yes | Yes (more aggressive) |
| Obfuscation | Yes | Yes |
| Optimization | Yes | Yes (more aggressive) |
| Kotlin support | Limited | Enhanced |
| Build speed | Slower | Faster |
| Java 8+ features | No | Yes (with desugaring) |
| Configuration | Complex | Simpler (uses Proguard rules) |

###### R8 Specific Rules

While R8 uses Proguard rules, there are some R8-specific rules you can use:

```
# Keep the class but allow its methods to be removed if unused
-keepclassmembers class com.example.MyClass { *; }

# Keep the class and all its members, but allow optimization
-keepclasseswithmembers class com.example.MyClass { *; }

# R8 specific: Allow aggressive inlining of a method
-alwaysinline class com.example.MyClass { 
    void myMethod(); 
}

# R8 specific: Prevent inlining of a method
-neverinline class com.example.MyClass { 
    void myMethod(); 
}
```

###### Debugging R8 Issues

When troubleshooting issues with R8, these approaches can be helpful:

1. **Inspect the R8 output**: Use the `--debug` flag to see what R8 is doing:

```
android.buildTypes.release.proguardFiles += file("debug-rules.pro")
```

With debug-rules.pro containing:
```
-printusage unused.txt
-printseeds seeds.txt
-printmapping mapping.txt
-verbose
```

2. **Disable specific optimizations**: If you suspect a particular optimization is causing issues:

```
-optimizations !class/merging/vertical,!field/*,!method/inlining/*
```

3. **Use the R8 rewriter tool**: For deobfuscating stack traces from production crashes:

```
java -jar r8.jar retrace --mapping mapping.txt obfuscated_stacktrace.txt
```

---

#### Resource Shrinking and Optimization

Resources often account for a significant portion of app size. Here are strategies to optimize them:

##### Enable Resource Shrinking

Resource shrinking removes unused resources from your packaged app. You can enable it in your build.gradle file by setting shrinkResources to true along with minifyEnabled.

##### Image Optimization

1. **Use WebP format**: Convert PNG images to WebP for better compression

   WebP provides superior compression compared to PNG and JPEG while maintaining good quality. You can convert your images using Android Studio:

   - Right-click on a drawable resource
   - Select "Convert to WebP"
   - Configure the encoding options (lossy or lossless)
   - Review the size savings and quality

   For batch conversion, you can use the command-line tool:

   ```
   cwebp -q 80 input.png -o output.webp
   ```

2. **Vector drawables**: Use vector drawables instead of multiple PNG files for different densities:

```xml
<!-- vector_drawable.xml -->
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="24dp"
    android:height="24dp"
    android:viewportWidth="24"
    android:viewportHeight="24">
    <path
        android:fillColor="#FF000000"
        android:pathData="M12,2C6.48,2 2,6.48 2,12s4.48,10 10,10 10,-4.48 10,-10S17.52,2 12,2z"/>
</vector>
```

   Vector drawables are resolution-independent and typically much smaller than their bitmap counterparts. To ensure compatibility with older Android versions, add this to your build.gradle:

   ```
   android {
       defaultConfig {
           vectorDrawables.useSupportLibrary = true
       }
   }
   ```

3. **Density-specific resources**: Provide only necessary density resources by configuring resConfigs in your build.gradle file

   ```
   android {
       defaultConfig {
           // Include only the density buckets you need
           resConfigs "nodpi", "hdpi", "xhdpi", "xxhdpi"
       }
   }
   ```

4. **Compress PNG files**: Use tools like pngquant or TinyPNG to optimize PNG files without significant quality loss:

   ```
   pngquant --quality=65-80 image.png
   ```

5. **Lazy loading for large images**: Instead of bundling large images in your app, consider loading them on demand from a server.

6. **Drawable tinting**: Use a single grayscale image and apply tints programmatically instead of including multiple colored versions.

   ```xml
   <!-- In layout file -->
   <ImageView
       android:id="@+id/icon"
       android:layout_width="24dp"
       android:layout_height="24dp"
       android:src="@drawable/icon_grayscale"
       app:tint="@color/colorPrimary" />
   ```

##### Language Resources

Limit included languages to reduce APK size by configuring resConfigs in your build.gradle file to include only the languages your app supports:

```
android {
    defaultConfig {
        // Include only the languages you support
        resConfigs "en", "es", "fr", "de"
    }
}
```

This configuration removes resource files for unsupported languages, which can significantly reduce app size, especially for apps with many string resources.

##### Layout Optimization

1. **Flatten view hierarchies**: Simplify your layouts to reduce the number of nested ViewGroups:

   - Use ConstraintLayout instead of nested LinearLayouts
   - Avoid unnecessary container views
   - Use the Layout Inspector to identify overly complex view hierarchies

2. **Reuse layouts with <include>**: Extract common UI components into separate layout files and include them:

   ```xml
   <include layout="@layout/common_header" />
   ```

3. **Use styles and themes**: Define styles for common view attributes instead of repeating them:

   ```xml
   <style name="AppButton">
       <item name="android:background">@drawable/button_bg</item>
       <item name="android:textColor">@color/button_text</item>
       <item name="android:padding">8dp</item>
   </style>
   ```

4. **Remove unused resources**: Use the Lint tool to identify unused resources:

   - In Android Studio, go to Analyze > Inspect Code
   - Look for "Unused resources" in the results
   - Remove or mark resources as tools:keep if needed

---

#### Advanced Techniques

##### App Bundles and Dynamic Delivery

Android App Bundles (AAB) represent a significant improvement over traditional APKs for app distribution. They optimize delivery by providing only the necessary code and resources for each device. Here's why you should consider using App Bundles instead of APKs:

1. **Smaller download sizes**: App Bundles generate optimized APKs for each device configuration, reducing download sizes by up to 15% compared to a universal APK. Users only download the code and resources they need for their specific device.

2. **Automatic language resources optimization**: App Bundles deliver only the language resources that match the user's device settings, eliminating unnecessary translations.

3. **Screen density optimization**: Only the appropriate screen density resources are delivered to each device, avoiding the inclusion of unused drawable resources.

4. **Architecture-specific code delivery**: Users receive only the native libraries that match their device's CPU architecture (ARM, ARM64, x86, etc.) rather than all architectures.

5. **Dynamic feature delivery**: App Bundles enable on-demand delivery of features that aren't needed for the initial app launch, further reducing initial download size.

6. **Simplified app publishing**: Developers maintain a single artifact for all device configurations, with Google Play handling the generation of optimized APKs.

7. **Play Feature Delivery**: Enables advanced delivery options like conditional delivery, instant delivery, and on-demand delivery of app features.

8. **Easier updates**: Smaller update packages as users only need to download changes relevant to their device configuration.

You can configure language, density, and ABI splits in your build.gradle file to take full advantage of App Bundles:

##### On-demand Features with Dynamic Feature Modules

Split your app into base and feature modules that can be downloaded on demand. This requires configuring dynamic feature modules in your project.

##### Native Library Optimization

Reduce the size of native libraries by including only specific ABIs and enabling native library compression in your build.gradle file.

---

#### Measuring and Monitoring App Size

To effectively optimize app size, you need to measure and monitor it throughout development:

1. **APK Analyzer**: Use Android Studio's built-in APK Analyzer to inspect your APK contents
2. **Custom Gradle task**: Create a task to report app size after each build
3. **CI integration**: Add app size checks to your CI pipeline

You can create a custom Gradle task to report app size after each build, which helps track size changes over time.

---

### Conclusion

Reducing app size is a multifaceted challenge that requires attention to code, resources, and delivery mechanisms. By implementing the techniques discussed in this post—code shrinking with Proguard and R8, resource optimization, App Bundles, and other advanced delivery mechanisms—you can significantly reduce your app's size while maintaining or even improving its performance.

Remember that app size optimization is an ongoing process. Regularly analyze your app's composition, monitor size changes with each release, and continuously refine your optimization strategies. Migrating from traditional APKs to App Bundles can provide immediate benefits with minimal effort, making it one of the most effective steps you can take. The result will be a more efficient app that provides a better user experience through faster downloads, reduced storage requirements, and improved runtime performance.
