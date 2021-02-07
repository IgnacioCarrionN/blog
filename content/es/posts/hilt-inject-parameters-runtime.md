---
author: "Ignacio Carrión"
title: "Hilt: Inyectar valores al ViewModel en tiempo de ejecución."
date: 2021-02-07T10:00:06+01:00
description: "First post in my new Kotlin and Android development blog"
hideToc: false
enableToc: true
enableTocContent: false
author: Ignacio
authorEmoji: 🤖
image: images/kotlin/kotlin-logo.png
draft:true
tags: 
- kotlin
- android
- jetpack
- coroutines
- androidx
---
# Inyección de valores en tiempo de ejecución con Dagger-Hilt

Desde que apareció Hilt para facilitar la inyección de dependencias en aplicaciones Android, no era posible la inyección de dependencias en tiempo de ejecución sin utilizar librerías ajenas a Dagger o Hilt. Desde la versión 2.31 se incorpora en Dagger la anotación `@AssistedInject`. Con esta anotación vamos a ser capaces de indicar a Dagger-Hilt que dependencias se tienen que resolver en tiempo de ejecución y retrasar la inyección de esos parámetros hasta tener los valores. 

Esto era necesario para poder inyectar valores en los constructores de los ViewModel y poder ejecutar alguna operación en el método `init` del mismo. Como puede ser una petición a una API externa o bien una consulta en la base de datos local.

En este artículo veremos como implementar el `@AssistedInject` de Dagger para la inyección de valores en tiempo de ejecución en ViewModels con Hilt.

## Instalación

 En el fichero `build.gradle` raíz del proyecto, incluiremos el siguiente classpath: 
``` groovy 
classpath 'com.google.dagger:hilt-android-gradle-plugin:2.31.2-alpha'
```
Una vez añadido el *classpath* añadiremos el plugin de Hilt en el fichero `build.gradle` del módulo *app*.

``` groovy
apply plugin: 'dagger.hilt.android.plugin'
```

Y también las siguientes líneas a nuestras dependencias:

``` groovy
implementation 'com.google.dagger:hilt-android:2.31.2-alpha'  
kapt 'com.google.dagger:hilt-android-compiler:2.31.2-alpha'
```

En este enlace puedes ver un ejemplo de un archivo `build.gradle` completo: [app/build.gradle](https://github.com/IgnacioCarrionN/HiltAssistedInject/blob/master/app/build.gradle)

## Implementación

Para este ejemplo usaremos una clase repositorio encargada de recibir el nombre de usuario y devolver un mensaje de bienvenida. Para ello crearemos la siguiente interfaz:

``` kotlin
interface UserRepository {  
    fun getMessage(name: String): String  
}
```

Y su implementación: 

``` kotlin
class UserRepositoryImpl @Inject constructor() : UserRepository {  
    override fun getMessage(name: String): String {  
        return "Hi $name"  
  }  
}
```
Anotamos el constructor con `@Inject` para posteriormente poder declarar un `@Binds` en el módulo de Hilt e inyectar la implementación cada vez que se pida una interfaz del tipo *UserRepository*.

Vamos a crear el siguiente `ViewModel` que será el encargado de recibir el nombre del usuario desde el `Activity` o `Fragment` y llamar al repositorio para recibir el mensaje de bienvenida:

``` kotlin
class UserViewModel @AssistedInject constructor(  
    private val repository: UserRepository,  
    @Named("UserDispatcher") private val dispatcher: CoroutineDispatcher,  
    @Assisted private val name: String  
) : ViewModel() {  
  
    ...
  
}
```

En este `ViewModel` podemos ver como se anota el constructor con `@AssistedInject` para indicar a Dagger-Hilt que esta clase contiene dependencias que se deben inyectar en tiempo de ejecución. Esas dependencias están anotadas con `@Assisted`.

Para poder crear el `ViewModel` con la extensión `by viewModels()` de la librería de AndroidX debemos crear la `Factory` que más tarde pasaremos a la extensión:

``` kotlin
class Factory(  
    private val assistedFactory: UserViewModelAssistedFactory,  
    private val name: String,  
) : ViewModelProvider.Factory {  
    override fun <T : ViewModel?> create(modelClass: Class<T>): T {  
        return assistedFactory.create(name) as T  
    }
}
```

Con esto ya podemos completar nuestro ViewModel con la lógica necesaria para pedir la respuesta al repositorio y exponer al `Fragment` o `Activity` a través de un StateFlow.

El ViewModel completo quedaría:

``` kotlin
class UserViewModel @AssistedInject constructor(  
    private val repository: UserRepository,  
    @Named("UserDispatcher") private val dispatcher: CoroutineDispatcher,  
    @Assisted private val name: String  
) : ViewModel() {

    class Factory(  
        private val assistedFactory: UserViewModelAssistedFactory,  
        private val name: String,  
    ) : ViewModelProvider.Factory {  
        override fun <T : ViewModel?> create(modelClass: Class<T>): T {  
            return assistedFactory.create(name) as T  
        }
    }  
  
    private val _message: MutableStateFlow<String> = MutableStateFlow("")  
    val message: StateFlow<String>  
        get() = _message  
    
    init {  
        viewModelScope.launch(dispatcher) {  
            _message.emit(repository.getMessage(name))  
        }  
    }  

}
```
Relativo a `Hilt` solo nos faltaría declarar el módulo indicando como proveer las dependencias. Para este ejemplo usaremos el siguiente módulo:

``` kotlin
@Module  
@InstallIn(ActivityComponent::class)  
abstract class MainModule {  
  
    companion object {  
        @Provides  
        @Named("UserDispatcher")  
        fun provideUserDispatcher(): CoroutineDispatcher = Dispatchers.IO  
    }  
  
    @Binds  
    abstract fun provideUserRepository(repositoryImpl: UserRepositoryImpl): UserRepository  
}
```

En este módulo declaramos un `Dispatcher` para que sea más sencillo testear este ViewModel en un futuro. Y hacemos `@Binds` de nuestra interfaz con su implementación.

Ahora podemos inyectar nuestro repositorio en una `Activity` o `Fragment` de la siguiente forma:

``` kotlin
private val navArgs: UserFragmentArgs by navArgs()

@Inject  
lateinit var assistedFactory: UserViewModelAssistedFactory  
  
private val userViewModel: UserViewModel by viewModels {  
    UserViewModel.Factory(assistedFactory, navArgs.name)  
}
```
Simplemente nos faltaría observar los cambios en el StateFlow del ViewModel para poder actualizar nuestra UI. Eso se haría de la siguiente manera en un `Fragment` aunque sería muy similar en un `Activity`

``` kotlin
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {  
    super.onViewCreated(view, savedInstanceState)  
    lifecycleScope.launchWhenStarted {  
        userViewModel.message.collect {  
            binding.name.text = it  
        } 
    }
}
```

## Conclusión

Como hemos podido observar con `@AssistedInject` de Dagger podemos inyectar valores en tiempo de ejecución de una forma sencilla y podemos seguir utilizando los `navArgs` de AndroidX.

En el siguiente repositorio teneis el ejemplo completo: [HiltAssistedInject](https://github.com/IgnacioCarrionN/HiltAssistedInject)

En el próximo artículo hablaremos sobre como podemos hacer tests instrumentales a `Fragment` que usan Hilt.
