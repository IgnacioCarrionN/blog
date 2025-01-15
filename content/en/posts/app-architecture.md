---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Exploring App Architectures in Kotlin"
date: 2025-01-15T08:00:00+01:00
description: "Exploring App Architectures in Kotlin: MVC, MVP, MVVM, and MVI"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/mvvm.png
draft: false
tags: 
- kotlin
- architecture
---

### **Exploring App Architectures in Kotlin: MVC, MVP, MVVM, and MVI**

#### **Introduction**
In modern app development, choosing the right architecture is essential for creating maintainable and scalable applications. Architectures define how your codebase is organized and how different components interact. In this post, we’ll explore four popular app architectures: Model-View-Controller (MVC), Model-View-Presenter (MVP), Model-View-ViewModel (MVVM), and Model-View-Intent (MVI). We’ll look at their structure, pros, cons, and practical examples in Kotlin.

---

#### **1. Model-View-Controller (MVC)**

**Definition:**
MVC divides an app into three components:
- **Model:** Manages the data and business logic.
- **View:** Displays data to the user, directly accessing the Model for updates.
- **Controller:** Handles user input and updates the Model.

**Pros:**
- Simple to implement and understand.
- Effective for small apps or prototypes.

**Cons:**
- Tight coupling between View and Model.
- Limited separation of concerns; scaling can be challenging.

**Kotlin Example:**
```kotlin
// Model
data class User(var name: String, var age: Int)

// View
class UserView {
    fun displayUser(user: User) {
        println("Name: ${user.name}, Age: ${user.age}")
    }
}

// Controller
class UserController(private val model: User, private val view: UserView) {
    fun handleUserInput() {
        println("Enter new name for the user:")
        val newName = readLine() ?: ""
        model.name = newName  // Directly updates the model
        view.displayUser(model)
    }
}

fun main() {
    val user = User("Alice", 30)
    val view = UserView()
    val controller = UserController(user, view)
    view.displayUser(user)
    controller.handleUserInput()
}
```

---

#### **2. Model-View-Presenter (MVP)**

**Definition:**
In MVP, the Presenter mediates between the Model and View. Unlike MVC, the View is passive and delegates all interaction logic to the Presenter, which retrieves data from the Model and updates the View.

**Pros:**
- Better separation of concerns compared to MVC.
- Easier to test since the Presenter handles all logic.

**Cons:**
- Presenter classes can become large ("God classes").
- Managing lifecycle events can be challenging.

**Kotlin Example:**
```kotlin
// Model
data class User(val name: String, val age: Int)

// View Interface
interface UserView {
    fun displayUser(name: String, age: Int)
}

// Presenter
class UserPresenter(private val view: UserView) {
    private var user = User("Bob", 25)

    fun loadUser() {
        view.displayUser(user.name, user.age)
    }

    fun updateUser() {
        println("Enter new name for the user:")
        val newName = readLine() ?: ""
        user = user.copy(name = newName)
        view.displayUser(user.name, user.age)
    }
}

// View Implementation
class ConsoleUserView : UserView {
    override fun displayUser(name: String, age: Int) {
        println("Name: $name, Age: $age")
    }
}

fun main() {
    val view = ConsoleUserView()
    val presenter = UserPresenter(view)
    presenter.loadUser()
    presenter.updateUser()
}
```

---

#### **3. Model-View-ViewModel (MVVM)**

**Definition:**
MVVM promotes a reactive approach. The ViewModel provides data to the View and reacts to changes in the Model. It often uses LiveData or Kotlin’s StateFlow.

**Pros:**
- Encourages clean separation of concerns.
- Excellent for reactive programming using coroutines or flows.

**Cons:**
- Requires familiarity with reactive paradigms.
- Data binding or state management can add complexity.

**Kotlin Example:**
```kotlin
// Model
data class User(val name: String, val age: Int)

// ViewModel
class UserViewModel {
    private val _user = MutableStateFlow(User("Charlie", 28))
    val user = _user.asStateFlow()

    fun updateUser(name: String) {
        _user.value = _user.value.copy(name = name)
    }
}

// View
class UserView(private val viewModel: UserViewModel) {
    fun render() {
        viewModel.user.collect { user ->
            println("Name: ${user.name}, Age: ${user.age}")
        }
    }

    fun getUserInput(): String {
        println("Enter new name for the user:")
        return readLine() ?: ""
    }

    fun updateUserName() {
        val newName = getUserInput()
        viewModel.updateUser(newName)
    }
}

fun main() = runBlocking {
    val viewModel = UserViewModel()
    val view = UserView(viewModel)
    view.render()
    view.updateUserName()
}
```

---

#### **4. Model-View-Intent (MVI)**

**Definition:**
MVI uses unidirectional data flow. The View sends user intents, the Model processes them, and the state is updated and rendered by the View.

**Pros:**
- Predictable state management.
- Encourages immutability and clear data flow.

**Cons:**
- Steeper learning curve.
- Overhead for simple apps.

**Kotlin Example:**
```kotlin
// Model
data class UserState(val name: String = "", val age: Int = 0)

// Intent
sealed class UserIntent {
    object LoadUser : UserIntent()
    data class UpdateUser(val name: String) : UserIntent()
}

// Reducer
fun userReducer(currentState: UserState, intent: UserIntent): UserState {
    return when (intent) {
        is UserIntent.LoadUser -> UserState(name = "Dave", age = 40)
        is UserIntent.UpdateUser -> currentState.copy(name = intent.name)
    }
}

// ViewModel
class UserViewModel {
    private val _state = MutableStateFlow(UserState())
    val state: StateFlow<UserState> = _state

    fun processIntent(intent: UserIntent) {
        _state.update { currentState -> userReducer(currentState, intent) }
    }
}

// View
class UserView(private val viewModel: UserViewModel) {
    fun render() {
        viewModel.state.collect { state ->
            println("Name: ${state.name}, Age: ${state.age}")
        }
    }

    fun sendIntent(intent: UserIntent) {
        viewModel.processIntent(intent)
    }
}

fun main() = runBlocking {
    val viewModel = UserViewModel()
    val view = UserView(viewModel)
    view.sendIntent(UserIntent.LoadUser)
    view.render()
    println("Enter new name for the user:")
    val newName = readLine() ?: ""
    view.sendIntent(UserIntent.UpdateUser(newName))
}
```

---

#### **Conclusion**
Each architecture has its strengths and trade-offs:
- **MVC:** Best for small, simple apps.
- **MVP:** Balances structure and simplicity.
- **MVVM:** Ideal for reactive programming.
- **MVI:** Great for predictable and scalable state management.

Consider your project’s complexity and requirements when choosing an architecture. Which one do you prefer?
