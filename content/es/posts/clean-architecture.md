---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Clean Architecture en Kotlin & Android"
date: 2025-01-18T08:00:00+01:00
description: "Clean Architecture en Kotlin & Android con ejemplos prácticos"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/domain-layer.png
draft: false
tags: 
- kotlin
- architecture
---

# Clean Architecture en Kotlin & Android

## Introducción

Al desarrollar aplicaciones Android, mantener la escalabilidad y la legibilidad es crucial. Sin un enfoque arquitectónico claro, los proyectos pueden volverse difíciles de mantener a medida que crecen. Aquí es donde **Clean Architecture**, introducida por **Uncle Bob (Robert C. Martin)**, se vuelve invaluable. Esta arquitectura enfatiza la separación de responsabilidades, haciendo que el código sea más modular, testeable y mantenible.

## Entendiendo Clean Architecture

Clean Architecture está estructurada en **tres capas principales**, cada una con un rol específico:

- **Capa de Presentación**: Maneja la UI y las interacciones del usuario.
- **Capa de Dominio**: Contiene la lógica de negocio, casos de uso e interfaces de repositorio.
- **Capa de Datos**: Implementa los repositorios, maneja llamadas a APIs y operaciones de base de datos.

El principio central de Clean Architecture es la **dirección de las dependencias**: cada capa solo debe depender de las capas más cercanas al núcleo (dominio). Esto garantiza flexibilidad y escalabilidad.

## Estructura del Proyecto

Un proyecto con Clean Architecture en Kotlin típicamente sigue esta estructura:

```
com.example.app
│── presentation (ViewModels, UI, Estado)
│── domain (Casos de Uso, Interfaces de Repositorio, Modelos)
│── data (Implementaciones de Repositorio, Fuentes de Datos, APIs, DB)
```

Cada capa debe estar en un módulo o paquete separado, asegurando una correcta separación de responsabilidades.

## Modularización

Para mejorar aún más el mantenimiento y la escalabilidad, considera estructurar tu proyecto en **módulos de Gradle**. Esto garantiza una clara separación entre capas y promueve la reutilización.

Un proyecto modularizado con Clean Architecture podría seguir esta estructura:

```
com.example.app
│── app (Módulo principal de la aplicación)
│── feature-user
│   │── domain (Casos de Uso, Interfaces de Repositorio, Modelos)
│   │── data (Implementaciones de Repositorio, Fuentes de Datos, APIs, DB)
│   │── presentation (UI y ViewModels para funcionalidades de usuario)
│── core (Utilidades comunes, networking, helpers de base de datos)
```

Beneficios de la modularización:

- Tiempos de compilación más rápidos debido a la compilación aislada de módulos.
- Mejor encapsulación del código y separación de responsabilidades.
- Desarrollo y mantenimiento de funcionalidades más sencillo.
- Mayor facilidad para realizar pruebas unitarias en módulos independientes.

## Implementando Clean Architecture con Kotlin

### 1. **Capa de Dominio (Lógica de Negocio Central)**

La **capa de dominio** define la lógica de negocio y los casos de uso. No debe depender de ningún framework o librería externa, lo que la convierte en la parte más estable de la aplicación.

#### Ejemplo: Definir una Interfaz de Repositorio

```kotlin
interface UserRepository {
    suspend fun getUserById(id: String): User
}
```

#### Ejemplo: Caso de Uso

```kotlin
class GetUserByIdUseCase(private val userRepository: UserRepository) {
    suspend operator fun invoke(id: String): User {
        return userRepository.getUserById(id)
    }
}
```

### 2. **Capa de Datos (Implementación de Repositorios y Fuentes de Datos)**

La **capa de datos** proporciona implementaciones concretas de las interfaces de repositorio. Interactúa con APIs, bases de datos o almacenamiento local.

#### Ejemplo: Fuente de Datos

```kotlin
interface UserRemoteDataSource {
    suspend fun fetchUserById(id: String): User
}

class UserRemoteDataSourceImpl(private val api: UserApi) : UserRemoteDataSource {
    override suspend fun fetchUserById(id: String): User {
        return api.fetchUserById(id)
    }
}
```

#### Ejemplo: Implementación del Repositorio

```kotlin
class UserRepositoryImpl(private val remoteDataSource: UserRemoteDataSource) : UserRepository {
    override suspend fun getUserById(id: String): User {
        return remoteDataSource.fetchUserById(id)
    }
}
```

### 3. **Capa de Presentación (UI & ViewModel)**

La **capa de presentación** es responsable de la lógica de UI y la gestión de estados. Depende de la **capa de dominio**, pero no interactúa directamente con la **capa de datos**.

#### Ejemplo: ViewModel

```kotlin
class UserViewModel(private val getUserByIdUseCase: GetUserByIdUseCase) : ViewModel() {

    private val _user = MutableStateFlow<User?>(null)
    val user: StateFlow<User?> get() = _user.asStateFlow()

    fun loadUser(id: String) {
        viewModelScope.launch {
            _user.value = getUserByIdUseCase(id)
        }
    }
}
```

## Mejores Prácticas

1. **Mantén la Capa de Dominio Pura**: No debe depender de frameworks de Android.
2. **Usa Inyección de Dependencias**: Koin ayuda a gestionar las dependencias de manera limpia.
3. **Sigue la Regla de las Dependencias**: Las capas internas no deben depender de las externas.
4. **Separa Interfaces e Implementaciones de Repositorios**: Las interfaces van en la capa de dominio, las implementaciones en la capa de datos.
5. **Usa Fuentes de Datos**: Encapsula llamadas a APIs y bases de datos en clases dedicadas.
6. **Modulariza tu Código**: Usa módulos de Gradle para separar responsabilidades y mejorar los tiempos de compilación.

## Conclusión

Clean Architecture proporciona una forma robusta de estructurar aplicaciones Android. Al separar responsabilidades y aplicar dependencias claras, el código se vuelve más testeable y escalable. Usar **Koin** para la inyección de dependencias mejora aún más la mantenibilidad. Adoptar esta arquitectura, junto con **modularización**, resultará en una base de código más modular y resistente para tus proyectos en Kotlin.
