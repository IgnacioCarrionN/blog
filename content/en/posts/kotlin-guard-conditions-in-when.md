---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Guard conditions in when starting in Kotlin 2.1.0."
date: 2024-12-17T08:00:00+01:00
description: "New feature since Kotlin 2.1.0, guard conditions in when expressions"
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
# Guard conditions in when in Kotlin 2.1.0

One of the new features in `Kotlin 2.1.0` is the guard conditions on `when` expressions, this feature will bring some advantages like:
- Reduce nesting
- Avoid boilerplate
- Improve readability

## Enable the feature

This feature is in preview state, for this you need to enable it starting on `Kotlin 2.1.0`. In the file `build.gradle.kts` we should add the new piece of code inside the `kotlin` block:

``` kotlin
kotlin {
    compilerOptions {  
        freeCompilerArgs.add("-Xwhen-guards")  
    }
}
```

## Use guard conditions on `when` expression branches

For this example we will use a `sealed interface` to handle the responses from a remote service:

``` kotlin
sealed interface Response<out T> {  
    data object Loading : Response<Nothing>  
    data class Content <out T> (val data: T?) : Response<T>
    data class Error(val error: Exception) : Response<Nothing> 
}
```

Loading, Content and Error implement the Response interface to manage the different states of the response.

### Before new feature

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
Here you can see how the code to print the `Unknown error` needs to be in both branches, also it adds nested complexity that reduces the code readability.

### Using new Guard conditions

You must add `if` statement after the primary condition inside the when branch, see below:

``` kotlin
fun <T> Response<T>.handle() = when (this) {  
    is Response.Loading -> println("Loading")  
    is Response.Content if data != null -> println("Content ${data::class.simpleName}")  
    is Response.Error if error is IllegalStateException -> println("Handled error")  
    else -> println("Unknown error")  
}
```

This way the print statement with the text `Unkwon error` is used just once, also we remove the nested complexity.

If we need to check different conditions on the else branch, we can use an `else if` in case our response doesn't satisfy the previous conditions.

``` kotlin
fun <T> Response<T>.handle() = when (this) {  
    is Response.Loading -> println("Loading")  
    is Response.Content if data != null -> println("Content ${data::class.simpleName}")  
    is Response.Error if error is IllegalStateException -> println("Handled IllegalState")  
    else if this is Response.Error && this.error is NoSuchElementException -> println("Handle NoSuchElement")  
    else -> println("Unknown error")  
}
```

This example can be simplified using to branches with the same primary condition `is Response.Error`, in my opinion keeps the code more readable and simple:

``` kotlin
fun <T> Response<T>.handle() = when (this) {  
    is Response.Loading -> println("Loading")  
    is Response.Content if data != null -> println("Content ${data::class.simpleName}")  
    is Response.Error if error is IllegalStateException -> println("Handled IllegalState")  
    is Response.Error if error is NoSuchElementException -> println("Handle NoSuchElement")  
    else -> println("Unknown error")  
}
```


## Conclusion

With this new feature we can add new conditions without repeating code and will allow more concise `when` expressions. It's in preview state starting from `Kotlin 2.1.0` but seems like it will become stable soon.

Here is the link to the documentation where you can find what's new in Kotlin 2.1.0 included the guard conditions in `when` expressions [kotlinlang](https://kotlinlang.org/docs/whatsnew21.html#guard-conditions-in-when-with-a-subject)