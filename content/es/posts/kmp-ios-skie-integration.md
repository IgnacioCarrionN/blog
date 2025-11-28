---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "KMP fluido en iOS: Mejorando la experiencia con SKIE"
date: 2025-11-28T10:00:00+01:00
description: "Descubre cómo SKIE transforma la experiencia de Kotlin Multiplatform para desarrolladores iOS uniendo la semántica de Kotlin y Swift."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/swift-suspend.png
draft: false
tags: 
- kotlin-multiplatform
- ios
- swift
- skie
- kmp
---

### KMP fluido en iOS: Mejorando la experiencia con SKIE

Kotlin Multiplatform (KMP) es revolucionario para compartir lógica de negocio entre plataformas. Sin embargo, para los desarrolladores iOS que consumen este código compartido, la experiencia a veces puede sentirse extraña. Esto se debe a que KMP compila para iOS a través de la interoperabilidad con Objective-C, la cual no soporta nativamente algunas de las mejores características modernas de Kotlin (y Swift).

Aquí es donde entra **[SKIE](https://skie.touchlab.co/)** (Swift Kotlin Interface Enhancer), desarrollado y mantenido por **Touchlab**.

Vale la pena mencionar que JetBrains está trabajando en `swift-export` para mejorar nativamente la experiencia de desarrollo en iOS. Sin embargo, hasta que esa funcionalidad esté lista, herramientas como SKIE son la mejor opción.

---

#### La experiencia "Vanilla" de KMP en iOS

Cuando compilas Kotlin a un framework de iOS sin herramientas como SKIE, puedes encontrar varios puntos de fricción:

1.  **Sealed Classes**: En Kotlin, las `sealed classes` permiten sentencias `when` exhaustivas. En Objective-C (y por tanto en Swift vía KMP), estas se convierten en jerarquías de clases estándar. Pierdes la capacidad del compilador para asegurar que has manejado todos los casos en un `switch`.
2.  **Enums**: Similar a las sealed classes, los enums de Kotlin no siempre se mapean 1-a-1 a enums de Swift con el mismo poder y flexibilidad.
3.  **Coroutines**: Las funciones `suspend` de Kotlin se generan como funciones con completion handlers (callbacks) en Objective-C. Aunque Swift tiene `async/await`, puentear estos callbacks manualmente es tedioso y propenso a errores.
4.  **Flows**: Kotlin `Flow` es una herramienta potente de procesamiento de streams. En KMP estándar, aparece como un objeto genérico, requiriendo adaptadores para ser consumido cómodamente en Swift.

---

#### Cómo SKIE mejora la experiencia del desarrollador iOS

SKIE actúa como un puente que genera código Swift amigable sobre el header de Objective-C. No cambia el compilador de Kotlin; mejora la salida.

##### 1. Sealed Classes y Enums Exhaustivos
SKIE genera enums de Swift reales para tus `sealed classes` e `interfaces` de Kotlin. Esto significa que puedes usar la sentencia `switch` de Swift con chequeo exhaustivo completo.

```swift
// Swift code with SKIE
switch viewState {
case .loading:
    showLoader()
case .data(let content):
    showContent(content)
case .error(let message):
    showError(message)
}
```

Sin SKIE, tendrías que hacer casting de tipos o chequear `is`, a menudo necesitando un caso `default` que se traga errores futuros.

##### 2. Soporte Nativo para Async/Await
SKIE convierte automáticamente las funciones `suspend` de Kotlin en funciones `async throws` de Swift. Esto permite a los desarrolladores iOS usar el modelo de concurrencia moderno al que están acostumbrados.

```swift
// Swift code with SKIE
do {
    let data = try await repository.getData()
    updateUI(data)
} catch {
    handleError(error)
}
```

##### 3. Flows como AsyncSequences
Quizás una de las características más geniales es transformar Kotlin `Flow` en Swift `AsyncSequence`. Esto te permite iterar sobre streams de datos usando un simple bucle `for await`.

```swift
// Swift code with SKIE
for await state in viewModel.stateFlow {
    render(state)
}
```

---

#### Cómo configurar SKIE

Configurar SKIE es increíblemente sencillo. Se distribuye como un plugin de Gradle.

1.  Abre tu `build.gradle.kts` raíz o el `build.gradle.kts` de tu módulo compartido.
2.  Añade el plugin SKIE al bloque `plugins`:

```kotlin
plugins {
    kotlin("multiplatform") version "2.0.0" // or your version
    id("co.touchlab.skie") version "0.9.0" // Check for latest version
}
```

3.  Sincroniza tu proyecto Gradle.

¡Eso es todo! SKIE se enganchará automáticamente a la tarea de linkado de tu framework iOS y generará el código Swift mejorado.

Para configuración más avanzada, como deshabilitar features específicas o ajustar la generación de enums, puedes usar el bloque de configuración `skie`:

```kotlin
skie {
    features {
        group("com.example.library") {
            SealedInterop.enabled = true
        }
    }
}
```

---

#### Conclusión

Si estás construyendo un proyecto KMP que apunta a iOS, usar SKIE es casi obligatorio para una experiencia pulida. Respeta las diferencias de la plataforma mientras te permite compartir la máxima cantidad de lógica, manteniendo a tu equipo iOS feliz y tu codebase limpia.
