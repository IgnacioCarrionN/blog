---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Hilt: Inject Runtime parameters to ViewModels."
date: 2021-02-24T07:00:06+01:00
description: "Inject parameters to ViewModels at Runtime in Android."
hideToc: false
enableToc: true
enableTocContent: false
author: Ignacio
authorEmoji: 💉
image: images/kotlin/kotlin-logo.png
draft: false
tags: 
- kotlin
- android
- jetpack
- coroutines
- androidx
---
# Inject runtime parameters with Dagger-Hilt

Since Hilt appeared to make it easier the dependency injection in Android, it was impossible to inject runtime parameters without using third party libraries. Since Dagger version 2.31, exists the `@AssistedInject` annotation. With this annotation we can instruct Dagger-Hilt what dependencies need to be created at runtime and delay the injection of this parameters until we can provide those values.

This is necessary to inject parameters into ViewModel constructor and be able to execute some code in the `init` function. It can be an external API call or some query to our local database.

In this post we will learn how to use `@AssistedInject` from Dagger to inject runtime parameters to ViewModels with Hilt.

## Installation

In the root project `build.gradle` file, we will include the Hilt classpath:

``` groovy 
classpath 'com.google.dagger:hilt-android-gradle-plugin:2.31.2-alpha'
```

Once we have Hilt *classpath* we will add Hilt plugin to `build.gradle` file from *app* module.

``` groovy
apply plugin: 'dagger.hilt.android.plugin'
```

And the next lines to our dependencies block:

``` groovy
implementation 'com.google.dagger:hilt-android:2.31.2-alpha'  
kapt 'com.google.dagger:hilt-android-compiler:2.31.2-alpha'
```
> We should keep in mind that we need `kapt` plugin on our `build.gradle`. For this we will add this line with the rest of plugins in our `build.gradle` from app module:

``` groovy
apply plugin: 'kotlin-kapt'
```

Those were the needed dependencies to make Hilt work in our project. In this post we will use libraries that are not defined here.

In this link you can see the complete `build.gradle` file: [app/build.gradle](https://github.com/IgnacioCarrionN/HiltAssistedInject/blob/master/app/build.gradle)

## Implementation

For this example we will be using a repository class with a function which receives a name and returns a welcome message. To acomplish this we will create the interface below:

``` kotlin
interface UserRepository {  
    fun getMessage(name: String): String  
}
```

And it's implementation:

``` kotlin
class UserRepositoryImpl @Inject constructor() : UserRepository {  
    override fun getMessage(name: String): String {  
        return "Hi $name"  
  }  
}
```

We should annotate the constructor with `@Inject` so we can declare a `@Binds` annotation in the Hilt module to be able to inject the implementation when we call an interface of type *UserRepository*.

Next we will create our `ViewModel`, this class will receive the user name from the `Activity` or `Fragment` and call the repository to get the welcome message:

``` kotlin
class UserViewModel @AssistedInject constructor(  
    private val repository: UserRepository,  
    @Named("UserDispatcher") private val dispatcher: CoroutineDispatcher,  
    @Assisted private val name: String  
) : ViewModel() {  
  
    ...
  
}
```

In this `ViewModel` we can see how we should annotate the constructor with `@AssistedInject` so Dagger-Hilt knows this class has dependencies that will be injected at runtime. This runtime dependencies will be annotated with `@Assisted`.

To be able to create our `ViewModel` with the extension `by viewModels()` from AndroidX library, we should create the `Factory` class wich will be provided to the extension:

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

You can see we need an interface called `UserViewModelAssistedFactory`. This interface will handle the runtime parameters injected to the `ViewModel`:

``` kotlin
@AssistedFactory
interface UserViewModelAssistedFactory {

    fun create(name: String): UserViewModel

}
```
It's an interface with a `create` function. This function receive all the runtime parameters we want to inject in our ViewModel. In this example we only need a `name` parameter, but in case we need more parameters injected at runtime, they will be provided to this function.

With this we are able to complete our ViewModel with the logic to get the answer from the repository and expose it to the `Fragment` or `Activity` through a StateFlow.

The complete `ViewModel`:

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
Related to `Hilt` we only have to create the module to handle the creation of dependencies. For this example we will be using this module:

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

In this module we declare a function to provide a `Dispatcher` so it will be easier to test this ViewModel in a future. We declare a `@Binds` function so when we inject a `UserRepository` interface Hilt provides its implementation `UserRepositoryImpl`.

Now we can user our ViewModel in `Activities` or `Fragments`:

``` kotlin
private val navArgs: UserFragmentArgs by navArgs()

@Inject  
lateinit var assistedFactory: UserViewModelAssistedFactory  
  
private val userViewModel: UserViewModel by viewModels {  
    UserViewModel.Factory(assistedFactory, navArgs.name)  
}
```
We need to `@Inject` the AssistedFactory and use the UserViewModel.Factory to create our `ViewModel`.

From this step we only need to observe changes in the ViewModel StateFlow to be able to update our UI. This can be done in `Fragments` observing from the `onViewCreated`.

``` kotlin
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {  
    super.onViewCreated(view, savedInstanceState)  
    viewLifecycleOwner.lifecycleScope.launchWhenStarted {
        userViewModel.message.collect {
            binding.name.text = it
        }
    }
}
```
> Remember you need a class extending Application annotated with `@HiltAndroidApp` and each Activity or Fragment that uses injection with Hilt need to be annotated with `@AndroidEntryPoint`.

## Conclusion

Now we can inject Runtime values with Dagger `@AssistedInject` in a simple way and we can keep using `navArgs` from AndroidX.

You can see the complete example in this repository: [HiltAssistedInject](https://github.com/IgnacioCarrionN/HiltAssistedInject) 
