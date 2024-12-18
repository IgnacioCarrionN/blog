---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Exportar a Swift en KMP"
date: 2024-12-17T08:00:00+01:00
description: "Nueva funcionalidad en Kotlin 2.1.0, exportar directamente a swift desde Kotlin"
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

# Exportar a Swift en KMP

Empezando con la versión 2.1.0 podemos empezar a probar a exportar a Swift en Kotlin. Esta funcionalidad te permite exportar los módulos compartidos de Kotlin a Swift sin usar Objective-C. Esto mejorará la experiancia de los desarrolladores de iOS cuando usen módulos de KMP.

Actualmente el soporte básico incluye:

- Exportar múltiples módulos de Gradle a swift.
- Definir los nombres de los módulos swift.
- Simplificar la estructura de paquetes.

## Activar la funcionalidad

Para empezar a probar esta funcionalidad debes activarla en el fichero `gradle.properties`:

```
kotlin.experimental.swift-export.enabled=true
```

## Configuración

Después de añadir la línea mostrada arriba necesitas añadir esta configuración al fichero `build.gradle.kts`:

```kotlin
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
El siguiente paso es configurar xcode para lanzar la nueva tarea `embedSwiftExportForXcode` en lugar de `embedAndSignAppleFrameworkForXcode`. Puedes realizar este cambio desde la configuración de `Build phases` de la iosApp desde xcode o bien desde Android Studio modificando el fichero `project.pbxproj`.

Debes cambiar esta línea:

`shellScript = "cd \"$SRCROOT/..\"\n./gradlew :shared:embedAndSignAppleFrameworkForXcode\n";`

Por esta otra:

`shellScript = "cd \"$SRCROOT/..\"\n./gradlew :shared:embedSwiftExportForXcode\n";`

Después de aplicar estos cambios deberías ser capaz de lanzar la aplicación de iOS desde Android Studio o desde xcode sin ningún problema.

## Antes de activar la funcionalidad

Si intentas navegar a la definición de una función de Kotlin desde xcode en un archivo swift, se mostrará el código Objective-C que se exporta del módulo compartido de Kotlin. Este fichero generado es enorme teniendo en cuenta la complejidad del projecto usado para este ejemplo.

Te voy a mostrar a continuación una pequeña pieza del archivo de 175 líneas generado desde el código de Kotlin:

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

Cuando activas al funcionalidad de exportar a Swift y compilas el proyecto, al intentar navegar a la definición de una función del código compartido de Kotlin, xcode te mostrará el código exportado de Swift.

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
El código mostrado arriba es el fichero completo con 28 líneas, una gran diferencia con las 175 líneas del código exportado en Objective-C. También es importante mencionar la menor complejidad y mayor legibilidad del código swift.

## Conclusion

Después de probar esta nueva funcionalidad, estoy realmente impresionado con la mejora que supone para el desarrollo de iOS en los proyectos KMP. También me sorprende la diferencia en el código exportado en Objective-C y swift. Estoy seguro que esta funcionalidad mejorará en las siguientes versiones y acercará la experiencia entre el desarrollo nativo y el desarrollo multiplataforma.

Puedes encontrar el repositorio con el código usado en este ejemplo en [SwiftExport](https://github.com/IgnacioCarrionN/KmpSwiftExport), con dos ramas, main, con la configuración del típico iOS framework conf, y la rama swift-export con la nueva funcionalidad habilitada.