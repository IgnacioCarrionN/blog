---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Explorando Arquitecturas de Apps en Kotlin"
date: 2025-01-15T08:00:00+01:00
description: "Explorando Arquitecturas de Apps en Kotlin: MVC, MVP, MVVM y MVI"
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/mvvm.png
draft: false
tags: 
- kotlin
- architecture
---

### **Explorando Arquitecturas de Apps en Kotlin: MVC, MVP, MVVM y MVI**

#### **Introducción**
En el desarrollo moderno de aplicaciones, elegir la arquitectura adecuada es esencial para crear aplicaciones mantenibles y escalables. Las arquitecturas definen cómo se organiza tu base de código y cómo interactúan los diferentes componentes. En este artículo, exploraremos cuatro arquitecturas populares: Model-View-Controller (MVC), Model-View-Presenter (MVP), Model-View-ViewModel (MVVM) y Model-View-Intent (MVI). Analizaremos su estructura, ventajas, desventajas y ejemplos prácticos en Kotlin.

---

#### **1. Model-View-Controller (MVC)**

**Definición:**
MVC divide una aplicación en tres componentes:
- **Model:** Gestiona los datos y la lógica de negocio.
- **View:** Muestra los datos al usuario, accediendo directamente al Model para actualizaciones.
- **Controller:** Maneja la entrada del usuario y actualiza el Model.

**Ventajas:**
- Simple de implementar y entender.
- Eficaz para aplicaciones pequeñas o prototipos.

**Desventajas:**
- Acoplamiento estrecho entre la View y el Model.
- Separación limitada de preocupaciones; escalar puede ser desafiante.

**Ejemplo en Kotlin:**
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

**Definición:**
En MVP, el Presenter actúa como mediador entre el Model y la View. A diferencia de MVC, la View es pasiva y delega toda la lógica de interacción al Presenter, quien obtiene datos del Model y actualiza la View.

**Ventajas:**
- Mejor separación de preocupaciones en comparación con MVC.
- Más fácil de probar, ya que el Presenter maneja toda la lógica.

**Desventajas:**
- Las clases de Presenter pueden volverse grandes ("clases Dios").
- Manejar eventos del ciclo de vida puede ser desafiante.

**Ejemplo en Kotlin:**
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

**Definición:**
MVVM promueve un enfoque reactivo. El ViewModel proporciona datos a la View y reacciona a los cambios en el Model. A menudo utiliza LiveData o StateFlow de Kotlin.

**Ventajas:**
- Fomenta una clara separación de preocupaciones.
- Excelente para programación reactiva utilizando corutinas o flujos.

**Desventajas:**
- Requiere familiaridad con paradigmas reactivos.
- El enlace de datos o la gestión de estados puede agregar complejidad.

**Ejemplo en Kotlin:**
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

**Definición:**
MVI utiliza un flujo de datos unidireccional. La View envía intenciones del usuario, el Model las procesa, y el estado se actualiza y es renderizado por la View.

**Ventajas:**
- Gestión de estado predecible.
- Fomenta la inmutabilidad y un flujo de datos claro.

**Desventajas:**
- Curva de aprendizaje pronunciada.
- Sobrecarga para aplicaciones simples.

**Ejemplo en Kotlin:**
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

#### **Conclusión**
Cada arquitectura tiene sus fortalezas y compromisos:
- **MVC:** Mejor para aplicaciones pequeñas y simples.
- **MVP:** Equilibra estructura y simplicidad.
- **MVVM:** Ideal para programación reactiva.
- **MVI:** Excelente para la gestión de estado predecible y escalable.

Considera la complejidad y los requisitos de tu proyecto al elegir una arquitectura. ¿Cuál prefieres tú?