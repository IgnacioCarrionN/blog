---
author: "Ignacio Carrión"
authorImage: "/images/bio/wilfred.png"
title: "Kotlin Data Structures: What to Use and When"
date: 2025-09-05T08:00:00+01:00
description: "A practical guide to Kotlin collections and core data structures — how they behave, when to use them, and their time complexities."
hideToc: false
enableToc: true
enableTocContent: false
image: images/kotlin/kotlin-data-structures.png
draft: false
tags: 
- kotlin
- collections
- data-structures
- algorithms
- complexity
---

### Kotlin Data Structures: What to Use and When

Choosing the right data structure is one of the most impactful performance decisions you can make. Kotlin gives you expressive, type‑safe APIs on top of the JVM’s mature collections — plus some Kotlin‑specific options like ArrayDeque and immutable collections via kotlinx.collections.immutable.

This guide focuses on practical choice making: what to use, when to use it, and what to expect in terms of time complexity.

---

#### TL;DR: Quick Recommendations

- Need ordered, indexable, resizable sequence with fast random access? Use MutableList (backed by ArrayList).
- Need fast membership checks with no duplicates and no particular order? Use HashSet.
- Need key→value lookup with no particular order? Use HashMap.
- Need insertion order preserved? Use LinkedHashSet / LinkedHashMap.
- Need priority by a score? Use PriorityQueue.
- Need fast queue/deque operations (ends)? Use ArrayDeque.
- Need sorted order and range queries? Use TreeSet / TreeMap (Java). 
- Need persistent/immutable collections for sharing across threads or layers? Consider kotlinx.collections.immutable.

---

#### Kotlin Collections 101

- Interfaces: List<T>, Set<T>, Map<K, V> with mutable counterparts MutableList, MutableSet, MutableMap.
- Default JVM implementations used by Kotlin:
  - MutableList → ArrayList
  - MutableSet → LinkedHashSet by default when built via mutableSetOf(), HashSet when requested explicitly
  - MutableMap → LinkedHashMap by default when built via mutableMapOf(), HashMap when requested explicitly
- Arrays: Kotlin Array<T> is a fixed‑size boxed array; primitive arrays (IntArray, LongArray, etc.) avoid boxing and are very efficient.

Tip: Program against interfaces (List, Set, Map) and choose the concrete type only where you construct the collection.

---

#### Lists: List and MutableList (ArrayList, LinkedList)

- ArrayList (default MutableList implementation)
  - Access by index: O(1)
  - Append at end: Amortized O(1)
  - Insert/remove at arbitrary index: O(n) (shifts elements)
  - Contains (by equals): O(n)
  - Iteration: O(n)
  - Memory: contiguous backing array; may over‑allocate for growth

- LinkedList (Java’s LinkedList if you deliberately choose it)
  - Access by index: O(n)
  - Add/remove at ends: O(1)
  - Insert/remove at iterator position: O(1) after navigation
  - Contains: O(n)
  - Higher per‑element memory overhead; worse cache locality

When to choose which:
- Prefer ArrayList for most cases: random access, appends, iteration.
- Consider LinkedList only when you do many insertions/removals in the middle via iterators and rarely random‑access by index.

Kotlin tips:
- Use buildList for convenient construction.
- For read‑only view over a mutable list, expose List not MutableList.

---

#### Arrays vs Lists

- Array<T>
  - Fixed size, O(1) get/set by index
  - Best when size is known and you need raw performance, or with primitive arrays (IntArray, etc.) to avoid boxing
- MutableList<T>
  - Resizable, easier APIs for insertion/removal

Rule of thumb: prefer MutableList unless you specifically need primitive arrays or fixed size.

---

#### Sets: HashSet, LinkedHashSet, TreeSet

- HashSet
  - add/remove/contains: Average O(1), Worst O(n)
  - No ordering guarantees
- LinkedHashSet
  - add/remove/contains: Average O(1)
  - Preserves insertion order; slightly more memory than HashSet
- TreeSet (Java)
  - add/remove/contains: O(log n)
  - Maintains sorted order via Comparable or Comparator

When to use:
- Use HashSet for fastest membership checks when order doesn’t matter.
- Use LinkedHashSet when you need stable iteration order (e.g., for UI consistency).
- Use TreeSet for sorted sets and range queries (headSet/tailSet/subSet).

---

#### Maps: HashMap, LinkedHashMap, TreeMap

- HashMap
  - put/get/remove: Average O(1), Worst O(n)
  - No ordering
- LinkedHashMap
  - put/get/remove: Average O(1)
  - Preserves insertion order (also supports access‑order with Java APIs)
- TreeMap (Java)
  - put/get/remove: O(log n)
  - Keys are sorted; supports range operations

When to use:
- Use HashMap for general fast lookups.
- Use LinkedHashMap when you need predictable iteration order.
- Use TreeMap for sorted keys, ceiling/floor, ranges.

---

#### Queues, Deques, Stacks

- ArrayDeque (Kotlin stdlib)
  - addFirst/addLast/removeFirst/removeLast/peek: O(1) amortized
  - Great for queue (FIFO) and deque use cases; also a better stack than java.util.Stack

- PriorityQueue (Java)
  - push/pop/peek: O(log n)
  - Retrieves elements by smallest/largest priority (min‑heap by default)

Kotlin tip: Use ArrayDeque for stack/queue semantics unless you specifically need priority ordering.

---

#### Sorted Collections

- Prefer TreeSet/TreeMap for always‑sorted collections with updates.
- For one‑off sorting of a list, use sorted(), sortedBy(), sortedWith() which are O(n log n).

---

#### Immutable and Persistent Collections

- Kotlin’s List/Set/Map interfaces have read‑only variants (e.g., List) but those are not deeply immutable — the underlying instance might still be mutable.
- For truly immutable, persistent data structures with structural sharing, use kotlinx.collections.immutable:
  - PersistentList, PersistentSet, PersistentMap
  - Typical operations are O(log n) with small constants; excellent for sharing across threads and undo/redo models.

Example:
```kotlin
val pl: PersistentList<Int> = persistentListOf(1, 2, 3)
val pl2 = pl.add(4) // returns a new list, original unchanged
```

---

#### Time Complexity Cheat Sheet

- ArrayList (MutableList default)
  - get/set: O(1)
  - addLast: amortized O(1)
  - add/remove at index: O(n)
  - contains: O(n)

- LinkedList
  - get/set by index: O(n)
  - add/remove at ends: O(1)
  - contains: O(n)

- HashSet / HashMap
  - add/remove/contains (set), put/get/remove (map): Avg O(1), Worst O(n)

- LinkedHashSet / LinkedHashMap
  - Same as hash variants with stable iteration order

- TreeSet / TreeMap
  - add/remove/contains / put/get/remove: O(log n)

- ArrayDeque
  - add/remove at ends: Amortized O(1)

- PriorityQueue
  - offer/poll/peek: O(log n)

- Persistent (immutable) collections
  - add/remove/update: Typically O(log n)

---

#### Choosing the Right Structure: Practical Scenarios

- You need to deduplicate items and check membership fast → HashSet.
- You need to keep insertion order for UI rendering → LinkedHashMap or LinkedHashSet.
- You need sorted keys with range operations → TreeMap.
- You need frequent random access by index → ArrayList.
- You need a FIFO queue or LIFO stack with high throughput → ArrayDeque.
- You need to repeatedly take the smallest/largest item → PriorityQueue with a Comparator.
- You need thread‑safe sharing without locks and clear state flow → kotlinx.collections.immutable.

---

#### Kotlin‑Specific Performance Tips

- Prefer interfaces in public APIs (List, Set, Map) to keep implementation swappable.
- Pre‑size collections when you know approximate size (e.g., ArrayList(capacity)) to reduce reallocation.
- Use primitive arrays (IntArray, etc.) for tight loops and large numeric data.
- Prefer sequence operations for large pipelines when you want laziness; prefer list operations when you want materialization and random access.
- Beware of boxing costs with generic collections of primitives.

---

#### Common Pitfalls

- Exposing MutableList/MutableMap from APIs leaks mutability; expose read‑only interfaces instead.
- Assuming List is immutable in Kotlin — it’s just read‑only view.
- Ignoring iteration order needs; HashMap/HashSet don’t guarantee it.
- Using LinkedList for random access workloads — it will be slow.

---

#### Final Thoughts

Pick the simplest structure that meets your semantic needs (ordering, uniqueness, key lookup) and only optimize further when profiling shows a real need. Kotlin’s standard collections plus a few extras like ArrayDeque and Persistent collections cover most application scenarios with excellent ergonomics and performance.
