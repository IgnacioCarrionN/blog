---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Estructuras de Datos en Kotlin: Qué usar y cuándo"
date: 2025-09-05T08:00:00+01:00
description: "Guía práctica de colecciones y estructuras de datos en Kotlin: cómo se comportan, cuándo usarlas y sus complejidades de tiempo."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/kotlin-data-structures.png
draft: false
tags: 
- kotlin
- colecciones
- estructuras-de-datos
- algoritmos
- complejidad
---

### Estructuras de Datos en Kotlin: Qué usar y cuándo

Elegir bien la estructura de datos es una de las decisiones de rendimiento más influyentes. Kotlin ofrece APIs expresivas y seguras en tipos sobre las colecciones del JVM, además de opciones específicas como ArrayDeque y colecciones inmutables vía kotlinx.collections.immutable.

Esta guía se centra en la elección práctica: qué usar, cuándo usarlo y qué esperar en términos de complejidad temporal.

---

#### TL;DR: Recomendaciones rápidas

- ¿Necesitas una secuencia ordenada, indexable, redimensionable y con acceso aleatorio rápido? Usa MutableList (respaldada por ArrayList).
- ¿Necesitas comprobaciones de pertenencia rápidas sin duplicados y sin orden específico? Usa HashSet.
- ¿Necesitas búsquedas clave→valor sin orden específico? Usa HashMap.
- ¿Necesitas preservar el orden de inserción? Usa LinkedHashSet / LinkedHashMap.
- ¿Necesitas prioridad por una puntuación? Usa PriorityQueue.
- ¿Necesitas operaciones rápidas de cola/deque (en los extremos)? Usa ArrayDeque.
- ¿Necesitas ordenación y consultas por rango? Usa TreeSet / TreeMap (Java).
- ¿Necesitas colecciones persistentes/inmutables para compartir entre hilos o capas? Considera kotlinx.collections.immutable.

---

#### Colecciones en Kotlin 101

- Interfaces: List<T>, Set<T>, Map<K, V> con contrapartes mutables MutableList, MutableSet, MutableMap.
- Implementaciones por defecto del JVM empleadas por Kotlin:
  - MutableList → ArrayList
  - MutableSet → LinkedHashSet por defecto con mutableSetOf(), HashSet cuando se solicita explícitamente
  - MutableMap → LinkedHashMap por defecto con mutableMapOf(), HashMap cuando se solicita explícitamente
- Arrays: Array<T> de Kotlin es de tamaño fijo y boxea los elementos; los arrays primitivos (IntArray, LongArray, etc.) evitan boxing y son muy eficientes.

Consejo: Programa contra interfaces (List, Set, Map) y elige el tipo concreto solo donde construyes la colección.

---

#### Listas: List y MutableList (ArrayList, LinkedList)

- ArrayList (implementación por defecto de MutableList)
  - Acceso por índice: O(1)
  - Añadir al final: O(1) amortizado
  - Insertar/eliminar en índice arbitrario: O(n) (desplaza elementos)
  - Contiene (por equals): O(n)
  - Iteración: O(n)
  - Memoria: array contiguo; puede sobreasignar para crecer

- LinkedList (LinkedList de Java si la eliges deliberadamente)
  - Acceso por índice: O(n)
  - Añadir/eliminar en extremos: O(1)
  - Insertar/eliminar en posición del iterador: O(1) tras navegar
  - Contiene: O(n)
  - Mayor sobrecarga de memoria por elemento; peor localidad de caché

Cuándo elegir cada una:
- Prefiere ArrayList en la mayoría de casos: acceso aleatorio, appends, iteración.
- Considera LinkedList solo cuando haces muchas inserciones/eliminaciones en mitad mediante iteradores y rara vez accedes por índice.

Consejos Kotlin:
- Usa buildList para construcción conveniente.
- Para exponer solo lectura sobre una lista mutable, expón List no MutableList.

---

#### Arrays vs Listas

- Array<T>
  - Tamaño fijo, get/set O(1) por índice
  - Ideal cuando el tamaño se conoce y necesitas máximo rendimiento, o con arrays primitivos (IntArray, etc.) para evitar boxing
- MutableList<T>
  - Redimensionable, APIs más cómodas para inserción/eliminación

Regla general: prefiere MutableList salvo que necesites arrays primitivos o tamaño fijo.

---

#### Conjuntos: HashSet, LinkedHashSet, TreeSet

- HashSet
  - add/remove/contains: Promedio O(1), Peor O(n)
  - Sin garantías de orden
- LinkedHashSet
  - add/remove/contains: Promedio O(1)
  - Preserva el orden de inserción; algo más de memoria que HashSet
- TreeSet (Java)
  - add/remove/contains: O(log n)
  - Mantiene ordenación por Comparable o Comparator

Cuándo usar:
- Usa HashSet para pertenencia rápida cuando el orden no importa.
- Usa LinkedHashSet cuando necesitas orden de iteración estable (p. ej., para UI consistente).
- Usa TreeSet para conjuntos ordenados y consultas por rango (headSet/tailSet/subSet).

---

#### Mapas: HashMap, LinkedHashMap, TreeMap

- HashMap
  - put/get/remove: Promedio O(1), Peor O(n)
  - Sin orden
- LinkedHashMap
  - put/get/remove: Promedio O(1)
  - Preserva el orden de inserción (también soporta orden por acceso con APIs de Java)
- TreeMap (Java)
  - put/get/remove: O(log n)
  - Claves ordenadas; soporta operaciones por rango

Cuándo usar:
- Usa HashMap para búsquedas rápidas generales.
- Usa LinkedHashMap cuando necesitas orden de iteración predecible.
- Usa TreeMap para claves ordenadas, ceiling/floor, rangos.

---

#### Colas, Deques, Pilas

- ArrayDeque (stdlib de Kotlin)
  - addFirst/addLast/removeFirst/removeLast/peek: O(1) amortizado
  - Excelente para colas (FIFO) y deques; también mejor pila que java.util.Stack

- PriorityQueue (Java)
  - push/pop/peek: O(log n)
  - Recupera por prioridad mínima/máxima (min-heap por defecto)

Consejo Kotlin: Usa ArrayDeque para semánticas de pila/cola salvo que necesites orden por prioridad.

---

#### Colecciones Ordenadas

- Prefiere TreeSet/TreeMap para colecciones siempre ordenadas con actualizaciones.
- Para ordenar una lista puntualmente, usa sorted(), sortedBy(), sortedWith() que son O(n log n).

---

#### Colecciones Inmutables y Persistentes

- Las interfaces List/Set/Map de Kotlin tienen variantes de solo lectura (p. ej., List) pero no son profundamente inmutables — la instancia subyacente puede ser mutable.
- Para inmutabilidad real y persistencia con sharing estructural, usa kotlinx.collections.immutable:
  - PersistentList, PersistentSet, PersistentMap
  - Operaciones típicas O(log n) con constantes pequeñas; excelentes para compartir entre hilos y modelos de deshacer/rehacer.

Ejemplo:
```kotlin
val pl: PersistentList<Int> = persistentListOf(1, 2, 3)
val pl2 = pl.add(4) // devuelve una nueva lista; la original no cambia
```

---

#### Hoja de Trucos de Complejidad Temporal

- ArrayList (MutableList por defecto)
  - get/set: O(1)
  - addLast: O(1) amortizado
  - add/remove en índice: O(n)
  - contains: O(n)

- LinkedList
  - get/set por índice: O(n)
  - add/remove en extremos: O(1)
  - contains: O(n)

- HashSet / HashMap
  - add/remove/contains (set), put/get/remove (map): Promedio O(1), Peor O(n)

- LinkedHashSet / LinkedHashMap
  - Igual que las variantes hash con orden de iteración estable

- TreeSet / TreeMap
  - add/remove/contains / put/get/remove: O(log n)

- ArrayDeque
  - add/remove en extremos: O(1) amortizado

- PriorityQueue
  - offer/poll/peek: O(log n)

- Colecciones persistentes (inmutables)
  - add/remove/update: Típicamente O(log n)

---

#### Elegir la Estructura Correcta: Escenarios Prácticos

- Necesitas deduplicar y comprobar pertenencia rápido → HashSet.
- Necesitas mantener el orden de inserción para renderizar UI → LinkedHashMap o LinkedHashSet.
- Necesitas claves ordenadas con operaciones por rango → TreeMap.
- Necesitas acceso aleatorio frecuente por índice → ArrayList.
- Necesitas una cola FIFO o pila LIFO de alto rendimiento → ArrayDeque.
- Necesitas extraer repetidamente el menor/mayor → PriorityQueue con Comparator.
- Necesitas compartir estado entre hilos sin bloqueos y con flujo claro → kotlinx.collections.immutable.

---

#### Consejos de Rendimiento Específicos de Kotlin

- Prefiere interfaces en APIs públicas (List, Set, Map) para poder cambiar la implementación.
- Pre-dimensiona cuando conozcas el tamaño aproximado (p. ej., ArrayList(capacity)) para reducir realocaciones.
- Usa arrays primitivos (IntArray, etc.) para bucles intensivos y datos numéricos grandes.
- Prefiere sequence para pipelines grandes cuando quieras pereza; prefiere listas cuando quieras materialización y acceso aleatorio.
- Ojo con el boxing al usar colecciones genéricas de primitivos.

---

#### Errores Comunes

- Exponer MutableList/MutableMap en APIs filtra mutabilidad; expón interfaces de solo lectura.
- Asumir que List es inmutable en Kotlin — es solo una vista de solo lectura.
- Ignorar necesidades de orden de iteración; HashMap/HashSet no lo garantizan.
- Usar LinkedList para cargas de trabajo de acceso aleatorio — será lenta.

---

#### Reflexión Final

Elige la estructura más simple que cumpla tus necesidades semánticas (orden, unicidad, búsqueda por clave) y optimiza más solo cuando el perfilado lo justifique. Las colecciones estándar de Kotlin, junto con extras como ArrayDeque y las colecciones persistentes, cubren la mayoría de escenarios con gran ergonomía y rendimiento.
