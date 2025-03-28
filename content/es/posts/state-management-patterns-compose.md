---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Patrones de Gestión de Estado en Jetpack Compose"
date: 2025-03-28T08:00:00+01:00
description: "Aprende patrones esenciales para gestionar el estado en aplicaciones Jetpack Compose, incluyendo estado inmutable, actualizaciones basadas en eventos y estrategias de testing"
hideToc: false
enableToc: true
enableTocContent: false
image: images/android/state-management-compose.png
draft: false
tags:
- android
- compose
- patrones
- estado
- pruebas
---

# Patrones de Gestión de Estado en Jetpack Compose

La gestión del estado es un aspecto crucial en el desarrollo de aplicaciones robustas y mantenibles con Jetpack Compose. Este artículo explora patrones esenciales y mejores prácticas para gestionar el estado de manera efectiva en tu UI con Compose, incluyendo estado inmutable, actualizaciones basadas en eventos y estrategias de pruebas.

## Entendiendo los Patrones de Gestión de Estado

La gestión efectiva del estado en Compose requiere entender cómo estructurar y manejar los cambios de estado de una manera mantenible, testeable y escalable. Esto involucra varios patrones clave:

1. **Clases de Estado Inmutable**: Define límites claros de estado y previene modificaciones no intencionadas
2. **Actualizaciones Basadas en Eventos**: Centraliza las modificaciones de estado a través de eventos bien definidos
3. **Flujo de Estado Predecible**: Asegura que los cambios de estado sigan un patrón consistente
4. **Arquitectura Testeable**: Estructura el código para facilitar pruebas exhaustivas

Veamos cada uno de estos patrones en detalle.

## 1. Fuente Única de la Verdad

La base de una gestión efectiva del estado es mantener una fuente única de la verdad para el estado de tu aplicación. Este patrón ayuda a prevenir inconsistencias y hace que los cambios de estado sean más predecibles.

El patrón de Fuente Única de la Verdad implica usar una interfaz/clase sellada para representar todos los posibles estados de tu UI. Este enfoque proporciona varios beneficios:

- Seguridad de tipos: El compilador asegura que manejes todos los estados posibles
- Consistencia: Todo el estado de la UI proviene de una fuente autoritativa
- Predecibilidad: Las transiciones de estado son explícitas y rastreables
- Mantenibilidad: Cada estado es una instantánea completa de la UI
- Testeable: Los cambios de estado pueden ser fácilmente verificados

Aquí te mostramos cómo implementar este patrón:

```kotlin
// Ejemplo del patrón Fuente Única de la Verdad
sealed interface UserUiState {
    object Loading : UserUiState
    data class Error(val message: String) : UserUiState
    data class Success(val user: User) : UserUiState
}

class UserViewModel : ViewModel() {
    private val _uiState = MutableStateFlow<UserUiState>(UserUiState.Loading)
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()

    fun loadUser(userId: String) {
        viewModelScope.launch {
            _uiState.emit(UserUiState.Loading)
            try {
                val user = userRepository.getUser(userId)
                _uiState.emit(UserUiState.Success(user))
            } catch (e: Exception) {
                _uiState.emit(UserUiState.Error(e.message ?: "Error desconocido"))
            }
        }
    }
}

// El Composable puede manejar fácilmente todos los estados en un solo lugar
@Composable
fun UserScreen(viewModel: UserViewModel) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when (val state = uiState) {
        is UserUiState.Loading -> LoadingSpinner()
        is UserUiState.Error -> ErrorMessage(state.message)
        is UserUiState.Success -> {
            UserContent(state.user)
        }
    }
}
```

## 2. Inmutabilidad del Estado

El segundo patrón clave es mantener objetos de estado inmutables. Este enfoque es crucial para:
- Prevenir condiciones de carrera en operaciones concurrentes
- Hacer que los cambios de estado sean explícitos y rastreables
- Permitir una detección eficiente de cambios en Compose
- Simplificar la depuración y el testing

Aquí te mostramos cómo implementar la gestión de estado inmutable:

```kotlin
// Define los modelos de dominio
data class SearchResult(
    val id: String,
    val title: String,
    val description: String
)

// Define una clase de estado inmutable
data class SearchState(
    val query: String = "",
    val results: List<SearchResult> = emptyList(),
    val selectedResult: SearchResult? = null,
    val isLoading: Boolean = false
)

class SearchViewModel : ViewModel() {
    private val _uiState = MutableStateFlow(SearchState())
    val uiState: StateFlow<SearchState> = _uiState.asStateFlow()

    fun onQueryChange(query: String) {
        _uiState.update { it.copy(
            query = query,
            isLoading = query.isNotEmpty()
        ) }
        searchResults(query)
    }

    fun onResultSelected(result: SearchResult) {
        _uiState.update { it.copy(selectedResult = result) }
    }

    private fun searchResults(query: String) {
        viewModelScope.launch {
            try {
                val results = searchRepository.search(query)
                _uiState.update { it.copy(
                    results = results,
                    isLoading = false
                ) }
            } catch (e: Exception) {
                _uiState.update { it.copy(
                    results = emptyList(),
                    isLoading = false
                ) }
            }
        }
    }
}

@Composable
fun SearchScreen(viewModel: SearchViewModel) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()

    Column(modifier = Modifier.padding(16.dp)) {
        SearchBar(
            query = state.query,
            onQueryChange = viewModel::onQueryChange
        )

        if (state.isLoading) {
            CircularProgressIndicator()
        } else {
            LazyColumn {
                items(
                    items = state.results,
                    key = { it.id }
                ) { result: SearchResult ->
                    SearchResultItem(
                        result = result,
                        isSelected = result == state.selectedResult,
                        onClick = { viewModel.onResultSelected(result) }
                    )
                }
            }
        }
    }
}
```

## 3. Actualizaciones Basadas en Eventos

El tercer patrón clave es usar eventos para hacer que las actualizaciones de estado sean predecibles. Este enfoque ayuda a:
- Centralizar la lógica de modificación de estado
- Hacer que las transiciones de estado sean explícitas y rastreables
- Asegurar que todas las actualizaciones de estado sigan un patrón consistente
- Simplificar el testing mediante la verificación del manejo de eventos

Aquí te mostramos cómo implementar actualizaciones basadas en eventos:

```kotlin
// Los eventos representan todas las formas posibles de modificar el estado
// Beneficios:
// 1. Seguridad de tipos: El compilador asegura que todos los eventos sean manejados
// 2. Centralizado: Todas las modificaciones de estado pasan por un único punto
// 3. Rastreable: Fácil de registrar y depurar cambios de estado
// 4. Testeable: Los eventos pueden ser fácilmente simulados y verificados
sealed interface ProfileEvent {
    object LoadProfile : ProfileEvent
    data class UpdateBio(val newBio: String) : ProfileEvent
    data class UpdateName(val newName: String) : ProfileEvent
    object SaveProfile : ProfileEvent
}

data class ProfileState(
    val name: String = "",
    val bio: String = "",
    val isLoading: Boolean = false,
    val isSaving: Boolean = false,
    val error: String? = null
)

class ProfileViewModel : ViewModel() {
    private val _uiState = MutableStateFlow(ProfileState())
    val uiState: StateFlow<ProfileState> = _uiState.asStateFlow()

    // Punto único de entrada para todas las modificaciones de estado
    // Esto asegura que:
    // 1. Todos los cambios de estado se manejen consistentemente
    // 2. Las modificaciones de estado sean fáciles de rastrear
    // 3. Los efectos secundarios se manejen adecuadamente
    fun onEvent(event: ProfileEvent) {
        when (event) {
            is ProfileEvent.LoadProfile -> loadProfile()
            is ProfileEvent.UpdateBio -> updateBio(event.newBio)
            is ProfileEvent.UpdateName -> updateName(event.newName)
            is ProfileEvent.SaveProfile -> saveProfile()
        }
    }

    // Cada manejador de eventos sigue el mismo patrón:
    // 1. Actualizar el estado para mostrar la operación en progreso
    // 2. Realizar la operación
    // 3. Actualizar el estado con el resultado o error
    private fun loadProfile() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true, error = null) }
            try {
                val profile = profileRepository.getProfile()
                _uiState.update { it.copy(
                    name = profile.name,
                    bio = profile.bio,
                    isLoading = false
                ) }
            } catch (e: Exception) {
                _uiState.update { it.copy(
                    isLoading = false,
                    error = e.message
                ) }
            }
        }
    }

    private fun updateBio(newBio: String) {
        _uiState.update { it.copy(bio = newBio) }
    }

    private fun updateName(newName: String) {
        _uiState.update { it.copy(name = newName) }
    }

    // Para operaciones asíncronas, manejamos los estados de carga y errores consistentemente
    private fun saveProfile() {
        viewModelScope.launch {
            _uiState.update { it.copy(isSaving = true, error = null) }
            try {
                profileRepository.saveProfile(
                    name = uiState.value.name,
                    bio = uiState.value.bio
                )
                _uiState.update { it.copy(isSaving = false) }
            } catch (e: Exception) {
                _uiState.update { it.copy(
                    isSaving = false,
                    error = e.message
                ) }
            }
        }
    }
}

@Composable
fun ProfileScreen(viewModel: ProfileViewModel) {
    // La capa UI solo necesita:
    // 1. Observar cambios de estado
    // 2. Enviar eventos al ViewModel
    // Esto crea una clara separación de responsabilidades
    val state by viewModel.uiState.collectAsStateWithLifecycle()

    Column(modifier = Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = state.name,
            onValueChange = { viewModel.onEvent(ProfileEvent.UpdateName(it)) },
            label = { Text("Nombre") }
        )

        OutlinedTextField(
            value = state.bio,
            onValueChange = { viewModel.onEvent(ProfileEvent.UpdateBio(it)) },
            label = { Text("Biografía") }
        )

        Button(
            onClick = { viewModel.onEvent(ProfileEvent.SaveProfile) },
            enabled = !state.isLoading && !state.isSaving
        ) {
            Text("Guardar Perfil")
        }

        state.error?.let { error: String ->
            Text(
                text = error,
                color = MaterialTheme.colorScheme.error
            )
        }

        if (state.isLoading) {
            CircularProgressIndicator(Modifier.align(Alignment.CenterHorizontally))
        }
    }
}
```

## 4. Estrategia de Pruebas

El cuarto patrón clave es implementar una estrategia de pruebas integral. Las pruebas de la gestión de estado en Compose involucran tres aspectos principales:
- Pruebas de actualizaciones y transiciones de estado del ViewModel
- Verificación del manejo de eventos y efectos secundarios
- Asegurar que la UI refleje correctamente los cambios de estado

Aquí te mostramos cómo implementar una estrategia de pruebas completa:

Para pruebas de ViewModel:
- El estado inicial es correcto
- Los eventos producen las actualizaciones de estado esperadas
- Las operaciones asíncronas manejan estados de carga y error
- Las actualizaciones de estado son atómicas y consistentes

Para pruebas de UI:
- Los componentes reflejan el estado actual
- Las interacciones del usuario disparan los eventos correctos
- Los estados de carga y error se muestran adecuadamente

Aquí tienes un ejemplo completo mostrando estos patrones de testing:

```kotlin
// La clase de prueba demuestra:
// 1. Configuración adecuada de pruebas con dependencias
// 2. Pruebas de diferentes flujos (éxito, error, actualizaciones inmediatas)
// 3. Verificación de transiciones de estado
class ProfileViewModelTest {
    private lateinit var viewModel: ProfileViewModel
    private lateinit var testRepository: TestProfileRepository

    @Before
    fun setup() {
        testRepository = TestProfileRepository()
        viewModel = ProfileViewModel(testRepository)
    }

    @Test
    fun `cargar perfil - flujo exitoso`() = runTest {
        // Dado
        val testProfile = Profile(name = "Usuario Test", bio = "Bio Test")
        testRepository.setProfile(testProfile)

        // Estado inicial
        assertThat(viewModel.uiState.value)
            .isEqualTo(ProfileState())

        // Cuando
        viewModel.onEvent(ProfileEvent.LoadProfile)

        // Entonces - Estado de carga
        assertThat(viewModel.uiState.value.isLoading).isTrue()
        assertThat(viewModel.uiState.value.error).isNull()

        // Completar trabajo asíncrono
        advanceUntilIdle()

        // Entonces - Estado de éxito
        with(viewModel.uiState.value) {
            assertThat(isLoading).isFalse()
            assertThat(name).isEqualTo(testProfile.name)
            assertThat(bio).isEqualTo(testProfile.bio)
            assertThat(error).isNull()
        }
    }

    @Test
    fun `cargar perfil - flujo de error`() = runTest {
        // Dado
        testRepository.setShouldError(true)

        // Cuando
        viewModel.onEvent(ProfileEvent.LoadProfile)
        advanceUntilIdle()

        // Entonces
        with(viewModel.uiState.value) {
            assertThat(isLoading).isFalse()
            assertThat(error).isNotNull()
        }
    }

    @Test
    fun `actualizar nombre actualiza el estado inmediatamente`() = runTest {
        // Cuando
        viewModel.onEvent(ProfileEvent.UpdateName("Nuevo Nombre"))

        // Entonces
        assertThat(viewModel.uiState.value.name).isEqualTo("Nuevo Nombre")
    }
}

// Tests de UI verifican el flujo completo:
// 1. Renderizado del estado inicial
// 2. Actualización de la UI con cambios de estado
// 3. Manejo de interacciones del usuario
@Test
fun `pantalla de perfil muestra indicador de carga y luego contenido`() {
    // Crear regla de test
    composeTestRule.setContent {
        ProfileScreen(viewModel = viewModel)
    }

    // Verificar estado inicial de carga
    composeTestRule.onNode(hasTestTag("loading")).assertIsDisplayed()

    // Completar la carga
    runTest {
        advanceUntilIdle()
    }

    // Verificar contenido
    composeTestRule.onNode(hasText("Usuario Test")).assertIsDisplayed()
    composeTestRule.onNode(hasText("Bio Test")).assertIsDisplayed()

    // Probar interacción
    composeTestRule.onNode(hasText("Guardar Perfil")).performClick()

    // Verificar que el evento de guardado fue manejado
    verify(viewModel).onEvent(ProfileEvent.SaveProfile)
}
```

La combinación de gestión de estado basada en eventos y estados inmutables hace que las pruebas sean sencillas:
- Los eventos proporcionan puntos de entrada claros para probar cambios de estado
- Los estados inmutables hacen que las aserciones sean simples y confiables
- Las transiciones de estado son fáciles de verificar
- Las pruebas de UI pueden centrarse en reflejar el estado y las interacciones del usuario

## Conclusión

La gestión efectiva del estado en Jetpack Compose requiere una combinación de patrones y mejores prácticas:

1. **Fuente Única de la Verdad**
   - Usa clases de estado inmutable para representar todos los estados posibles
   - Mantén la gestión de estado centralizada y predecible
   - Haz que los cambios de estado sean explícitos y rastreables

2. **Inmutabilidad del Estado**
   - Previene modificaciones no intencionadas con estado inmutable
   - Usa data classes con copy para actualizaciones de estado
   - Mantén la seguridad de hilos y la predecibilidad

3. **Actualizaciones Basadas en Eventos**
   - Define eventos claros para todas las modificaciones de estado
   - Centraliza la lógica de actualización de estado
   - Haz que las transiciones de estado sean explícitas y verificables

4. **Estrategia de Pruebas**
   - Prueba las transiciones de estado exhaustivamente
   - Verifica que las actualizaciones de UI reflejen los cambios de estado
   - Asegura el manejo adecuado de eventos

Siguiendo estos patrones, construirás aplicaciones Compose que son:
- Más mantenibles y fáciles de depurar
- Menos propensas a errores relacionados con el estado
- Más fáciles de probar y verificar
- Más escalables a medida que crece la complejidad
