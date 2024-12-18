---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Swift export in KMP"
date: 2024-12-18T08:00:00+01:00
description: "New feature in Kotlin 2.1.0, basic swift export from Kotlin"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/swift-export.png
draft: false
tags: 
- kotlin
- android
- kmp
---

# Swift export in Kmp

Starting from version 2.1.0 we can start testing the Swift export in Kotlin. This feature allows you to export the Kotlin shared modules to Swift without the use of Objective-C. This will improve the iOS developers experience when using KMP modules.

At the moment basic support includes:

- Export multiple Gradle modules to Swift.
- Define the Swift module names.
- Flatten package structure

## Enable the feature

To start testing this functionality you should enable it on `gradle.properties` file:

```
kotlin.experimental.swift-export.enabled=true
```

## Configuration

After adding the line above you need to add this configuration to the `build.gradle.kts` file:

``` kotlin
kotlin {
	iosX64()  
    iosArm64()  
    iosSimulatorArm64()  
  
    @OptIn(ExperimentalSwiftExportDsl::class)  
    swiftExport {
	    moduleName = "shared"
	    flattenPackage = "dev.carrion.kmpswiftexport"
	}
}
```

Next step is configuring xcode to launch the new task `embedSwiftExportForXcode` instead of `embedAndSignAppleFrameworkForXcode`. You can do it from xcode build phases configuration of the iosApp or from Android Studio modifying the `project.pbxproj` file.

You should change this line:

`shellScript = "cd \"$SRCROOT/..\"\n./gradlew :shared:embedAndSignAppleFrameworkForXcode\n";`

With this one:

`shellScript = "cd \"$SRCROOT/..\"\n./gradlew :shared:embedSwiftExportForXcode\n";`

After this changes you should be able to launch the app from Android Studio or xcode without any problems.


## Before enabling the feature

If you try to jump to the definition of a kotlin function from xcode in a swift file, you will be prompted with the Objective-C code exported from the kotlin shared module. This file is huge given the complexity of the project used for this example.

I will show you just a little piece of the 175 lines file generated from the Kotlin source code:

```objectivec
__attribute__((objc_subclassing_restricted))
__attribute__((swift_name("Greeting")))
@interface SharedGreeting : SharedBase
- (instancetype)init __attribute__((swift_name("init()"))) __attribute__((objc_designated_initializer));
+ (instancetype)new __attribute__((availability(swift, unavailable, message="use object initializers instead")));
- (NSString *)greet __attribute__((swift_name("greet()")));
@end

__attribute__((swift_name("Platform")))
@protocol SharedPlatform
@required
@property (readonly) NSString *name __attribute__((swift_name("name")));
@end

__attribute__((objc_subclassing_restricted))
__attribute__((swift_name("IOSPlatform")))
@interface SharedIOSPlatform : SharedBase <SharedPlatform>
- (instancetype)init __attribute__((swift_name("init()"))) __attribute__((objc_designated_initializer));
+ (instancetype)new __attribute__((availability(swift, unavailable, message="use object initializers instead")));
@property (readonly) NSString *name __attribute__((swift_name("name")));
@end

__attribute__((objc_subclassing_restricted))
__attribute__((swift_name("Platform_iosKt")))
@interface SharedPlatform_iosKt : SharedBase
+ (id<SharedPlatform>)getPlatform __attribute__((swift_name("getPlatform()")));
@end

#pragma pop_macro("_Nullable_result")
#pragma clang diagnostic pop
NS_ASSUME_NONNULL_END
```

## After enabling the feature

When you enable the feature and build the project, trying to go to the definition of a function from the Kotlin code, xcode will show you the exported Swift code.

```swift
@_exported import ExportedKotlinPackages
@_implementationOnly import SharedBridge_shared
import KotlinRuntime

public typealias Greeting = ExportedKotlinPackages.dev.carrion.kmpswiftexport.Greeting

public func getPlatform() -> Swift.Never {
	ExportedKotlinPackages.dev.carrion.kmpswiftexport.getPlatform()
}

public extension ExportedKotlinPackages.dev.carrion.kmpswiftexport {

	public final class Greeting : KotlinRuntime.KotlinBase {
		public override init() {
			let __kt = dev_carrion_kmpswiftexport_Greeting_init_allocate()
			super.init(__externalRCRef: __kt)
			dev_carrion_kmpswiftexport_Greeting_init_initialize__TypesOfArguments__Swift_UInt__(__kt)
		}
		
		public override init(
			__externalRCRef: Swift.UInt
		) {
			super.init(__externalRCRef: __externalRCRef)
		}
		
		public func greet() -> Swift.String {
			return dev_carrion_kmpswiftexport_Greeting_greet(self.__externalRCRef())
		}
	}
	
	public static func getPlatform() -> Swift.Never {
		fatalError()
	}
}
```

The code above this lines is the complete file with 28 lines, a huge difference with the 175 lines from the Objective-C exported code. Also it's important to mention the lower complexity and higher readability on the Swift example.

## Conclusion

After testing this new feature, I'm really amazed with the improvement it brings to the iOS development in KMP projects. Also impressed with the difference in code between Objective-C and Swift exported codes. I'm sure this feature will improve in next versions and it will close the bridge between the native and multiplatform development experiences.

You can find the repository with the code from this example in [SwiftExport](https://github.com/IgnacioCarrionN/KmpSwiftExport), with two branches, main, where it's the usual iOS framework configuration, and swift-export branch which has the new feature enabled.