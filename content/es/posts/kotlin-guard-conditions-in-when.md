---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Condiciones en las expresiones when para Kotlin 2.1.0."
date: 2024-12-17T08:00:00+01:00
description: "Nueva funcionalidad en Kotlin 2.1.0, condiciones en las expresiones when"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/guard-when-new.png
draft: false
tags: 
- kotlin
- android
- kmp
---
# Condiciones en las expresiones when en Kotlin 2.1.0

Una de las nuevas funcionalidades de `Kotlin 2.1.0` es las condiciones en las expresiones `when`, lo que tendría varias ventajas entre las que se incluye:
- Reducir anidaciones
- Evita código repetido
- Mejorar legibilidad

## Activar la funcionalidad en Kotlin 2.1.0

Esta funcionalidad se encuentra en preview lo que es necesario activarla explícitamente para poder usarla en `Kotlin 2.1.0`. En el fichero `build.gradle.kts` añadiremos el siguiente código dentro del bloque de `kotlin {}`:
``` kotlin
kotlin {
	compilerOptions {  
		freeCompilerArgs.add("-Xwhen-guards")  
	}
}
```


## Uso de condicionales dentro de las ramas de la expresiones `when`

Para este ejemplo usaremos una `sealed interface` para manejar respuestas de un servicio remoto:

``` kotlin
sealed interface Response<out T> {  
    data object Loading : Response<Nothing>  
    data class Content <out T> (val data: T?) : Response<T>
    data class Error(val error: Exception) : Response<Nothing> 
}
```
Esta interfaz la implementan Loading, Content y Error para gestionar los distintos estados de una respuesta.

### Antes de la nueva funcionalidad

``` kotlin
fun <T> Response<T>.handleOld() = when (this) {  
    is Response.Loading -> println("Loading")  
    is Response.Content -> if (data != null) {  
        println("Content ${data::class.simpleName}")  
    } else {  
        println("Unknown error")  
    }  
    is Response.Error ->  if (error is IllegalStateException) {  
        println("Handled error")  
    } else {  
        println("Unknown error")  
    }  
}
```

Como se puede ver en este caso se repite el código que muestra por pantalla `Unknown error` además de añadir anidaciones que dificultan la lectura del código.

### Usando los nuevos condicionales

Se debe añadir el `if` justo después de la condición primaria de la rama, por ejemplo:

``` kotlin
fun <T> Response<T>.handle() = when (this) {  
    is Response.Loading -> println("Loading")  
    is Response.Content if data != null -> println("Content ${data::class.simpleName}")  
    is Response.Error if error is IllegalStateException -> println("Handled error")  
    else -> println("Unknown error")  
}
```
De esta manera no se repite el código que mostraba por pantalla el texto de `Unknown error` y además eliminamos las anidaciones facilitando la lectura del código.

En caso de necesitar comprobar varias condiciones en la rama else se podría añadir una rama `else if` que controle el flujo de los casos que no cumplen las condiciones anteriores.

``` kotlin
fun <T> Response<T>.handle() = when (this) {  
    is Response.Loading -> println("Loading")  
    is Response.Content if data != null -> println("Content ${data::class.simpleName}")  
    is Response.Error if error is IllegalStateException -> println("Handled IllegalState")  
    else if this is Response.Error && this.error is NoSuchElementException -> println("Handle NoSuchElement")  
    else -> println("Unknown error")  
}
```
Esto último se puede simplificar usando dos ramas con la misma primera condición de `is Response.Error` que a mi parecer queda más simple:

``` kotlin
fun <T> Response<T>.handle() = when (this) {  
    is Response.Loading -> println("Loading")  
    is Response.Content if data != null -> println("Content ${data::class.simpleName}")  
    is Response.Error if error is IllegalStateException -> println("Handled IllegalState")  
    is Response.Error if error is NoSuchElementException -> println("Handle NoSuchElement")  
    else -> println("Unknown error")  
}
```


## Conclusión

Con esta nueva funcionalidad se podrá añadir nuevas condiciones sin tener que repetir código y permitirá que las expresiones `when` sean más concisas. En la versión `2.1.0` de Kotlin está en modo preview pero se espera que pronto esta nueva funcionalidad sea estable.

Aquí el enlace a la documentación con las novedades de Kotlin 2.1.0 donde se explica la funcionalidad de condicionales en las expresiones `when` [kotlinlang](https://kotlinlang.org/docs/whatsnew21.html#guard-conditions-in-when-with-a-subject)