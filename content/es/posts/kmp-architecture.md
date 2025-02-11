---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Mejores Prácticas de Arquitectura en Kotlin Multiplatform para Aplicaciones Móviles"
date: 2025-02-11T08:00:00+01:00
description: "Consejos de arquitectura para proyectos KMP usando Clean Architecture"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/expect-actual.png
draft: false
tags:
- kotlin
- compose
- cmp
- multiplatform
- cleancode
- architecture
---

# Mejores Prácticas de Arquitectura en Kotlin Multiplatform para Aplicaciones Móviles

Kotlin Multiplatform (KMP) permite a los desarrolladores compartir la lógica de negocio entre Android y iOS mientras mantienen implementaciones específicas de cada plataforma cuando es necesario. Estructurar un proyecto KMP de manera eficiente es clave para mantener la escalabilidad, testabilidad y aplicar Clean Architecture. En esta guía, exploraremos las mejores prácticas para diseñar una aplicación móvil con **Compose Multiplatform** y **Clean Architecture**.

---

## 1. **Estructura del Proyecto**

Una estructura de proyecto bien organizada mejora el mantenimiento y la separación de responsabilidades. Un enfoque común es seguir una **estructura multi-módulo**, ya sea con un único módulo compartido o con múltiples módulos basados en feature:

```
project-root/
 ├── core/
 │   ├── network/                  # Network shared logic
 │   │   ├── src/commonMain/       # Shared networking
 │   │   ├── src/androidMain/      # Android-specific implementations
 │   │   ├── src/iosMain/          # iOS-specific implementations
 ├── features/                 # Feature-based modules
 │   ├── feature1/             # Example feature module
 │   │   ├── domain/           # Domain layer (Use cases, repositories interfaces)
 │   │   ├── data/             # Data layer (Implementations, APIs, Database)
 │   │   ├── presentation/     # UI and ViewModels for Compose Multiplatform
 ├── composeApp/               # Application module integrating all features
 │   ├── src/commonMain/       # Can contain shared UI and navigation logic
 │   ├── src/androidMain/      # Android-specific implementations if needed
 │   ├── src/iosMain/          # iOS-specific implementations if needed
 ├── androidApp/               # Android application module
 ├── iosApp/                   # iOS application module
```

- **Feature Modules**: En lugar de un único módulo compartido, se pueden tener módulos compartidos por feature para mejorar la modularidad y escalabilidad. Estos pueden dividirse aún más en **domain, data y presentation** para una mejor separación de responsabilidades.
- **Core Modules**: Contiene utilidades compartidas como networking, logging y lógica de dominio común.
- **ComposeApp Module**: Actúa como el módulo principal de la aplicación, integrando todos los módulos de features y manejando la navegación, similar a un módulo `app` en un proyecto estándar de Android.

En la mayoría de los proyectos de **Compose Multiplatform**, el módulo `composeApp` se usa para ensamblar todas las features, gestionar la navegación y manejar otras preocupaciones a nivel de aplicación, similar al módulo `app` en un proyecto estándar de Android.

---

## 2. **Aplicando Clean Architecture en KMP**

Seguir **Clean Architecture** ayuda a mantener la separación de responsabilidades y mejorar la testabilidad. La arquitectura puede estructurarse en las siguientes capas:

### **Capa de Dominio (commonMain)**

- **Contiene la lógica de negocio** (Casos de uso, Interactors).
- **Define las interfaces de repositorio** para el acceso a datos.
- **No depende de ninguna implementación específica de la plataforma.**

```kotlin
interface UserRepository {
    suspend fun getUser(): User
}
```

### **Capa de Datos (commonMain, específica por plataforma)**

- **Implementa las interfaces de repositorio**.
- Usa `expect/actual` para APIs específicas de plataforma como networking, bases de datos, etc.
- Obtiene y procesa datos antes de exponerlos a la capa de dominio.

Ejemplo de `expect/actual` para un cliente HTTP:

```kotlin
expect class HttpClientProvider {
    fun getClient(): HttpClient
}
```

Implementación específica para Android:

```kotlin
actual class HttpClientProvider {
    actual fun getClient() = HttpClient(Android) {}
}
```

Implementación específica para iOS:

```kotlin
actual class HttpClientProvider {
    actual fun getClient() = HttpClient(Ios) {}
}
```

### **Capa de Presentación (Compose Multiplatform)**

Con **Compose Multiplatform**, podemos compartir componentes de UI entre plataformas mientras aprovechamos la renderización nativa. El módulo `composeApp` integra todos los módulos de features y maneja la navegación y la lógica a nivel de aplicación.

```kotlin
@Composable
fun UserScreen(viewModel: UserViewModel) {
    val user by viewModel.userState.collectAsState()
    Column(modifier = Modifier.padding(16.dp)) {
        Text("Hello, ${user?.name ?: "Guest"}", style = MaterialTheme.typography.h6)
    }
}
```

En **Android**, esto se renderiza con Jetpack Compose, y en **iOS**, con Compose para iOS.

---

## 3. **Gestión del Estado en KMP**

La gestión del estado en un proyecto KMP puede manejarse eficientemente con **StateFlow**.

```kotlin
class UserViewModel(private val repository: UserRepository) {
    private val _userState = MutableStateFlow<User?>(null)
    val userState: StateFlow<User?> = _userState

    fun loadUser() {
        viewModelScope.launch {
            _userState.value = repository.getUser()
        }
    }
}
```

Dado que Compose Multiplatform soporta `collectAsState()`, podemos observar y renderizar cambios de estado directamente en la UI.

---

## 4. **Testing en KMP**

- **Unit Tests en `commonTest`** usando `kotlin.test`.
- **Pruebas específicas de plataforma** en `androidTest` y `iosTest`.

Ejemplo de prueba unitaria compartida:

```kotlin
@Test
fun testUserRepository() = runTest {
    val repository = FakeUserRepository()
    assertNotNull(repository.getUser())
}
```

---

## **Conclusión**

Siguiendo estas mejores prácticas, puedes construir aplicaciones KMP escalables y mantenibles:

- **Usa una estructura de proyecto modularizada** con un módulo compartido o módulos por feature.
- **Sigue Clean Architecture** para mejorar la mantenibilidad.
- **Aprovecha Compose Multiplatform** para la UI, utilizando un módulo `composeApp` para integrar módulos de features y manejar la navegación.
- **Los módulos de features pueden dividirse aún más en dominio, datos y presentación** para mejorar la separación de responsabilidades.
- **Gestiona el estado eficientemente** con `StateFlow`.
- **Escribe pruebas completas** tanto en código compartido como en implementaciones específicas de plataforma.

KMP permite compartir código de manera eficiente mientras se preservan optimizaciones específicas de cada plataforma, lo que lo convierte en una opción poderosa para el desarrollo de aplicaciones móviles.

¿Te gustaría un repositorio en GitHub con un ejemplo de esta configuración? 🚀
