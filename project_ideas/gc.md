# Toy Garbage Collector in Java (Learning Project)

## Goal
Build a **mini managed runtime** inside Java:
- You allocate “objects” into a **toy heap** you control.
- You hand out **handles** (e.g., `int id`) instead of real Java references.
- Your GC reclaims unreachable toy objects.

This teaches GC fundamentals without needing JVM internals.

---

## What this is (and isn’t)

### ✅ What you can do in pure Java
- Implement a heap as arrays / lists / `ByteBuffer`.
- Define your own object format (headers + fields).
- Track “roots” explicitly (globals / simulated stack).
- Implement GC algorithms:
  - mark–sweep
  - mark–compact
  - copying (semi-space)
  - generational (later)

### ❌ What you can’t do from normal Java
- Replace or hook into the **JVM’s real GC** for actual Java objects.
- Reliably scan JVM stacks/registers or manage safepoints.

So: you’re building **a GC for your own heap**, not HotSpot’s heap.

---

## Core idea

### 1) Handles, not references
Your “objects” live in a structure you control (e.g., `ObjectRecord[] heap`).
When you allocate, you return a handle (index) like `int handle`.

### 2) Explicit roots
GC starts from known “root handles”, e.g.:
- a `List<Integer> roots` for globals
- a simulated call stack that pushes/pops handles

### 3) Graph reachability
Objects reference other objects via **handles** stored in their fields.
GC finds all objects reachable from roots.

---

## Data model (simple version)

### Object record
Each allocated object could be represented like:
- `marked`: boolean (mark bit)
- `refs`: `int[]` (handles to other objects)
- optional: `payload` (ints/bytes/whatever for fun)

### Heap
A fixed-size array is easiest:
- `ObjectRecord[] heap`
- free list or linear scan to find empty slots

---

## Milestones

### Milestone A: Minimal allocator
- `alloc(int refCount) -> int handle`
- store `ObjectRecord` in a free slot
- return its index as the handle

### Milestone B: Mark–sweep GC (first real collector)
**Mark phase**
- Start from each root handle.
- Traverse the object graph (DFS/BFS).
- Set `marked = true` on visited objects.

**Sweep phase**
- Scan entire heap:
  - if slot is occupied and `marked == false` → free it
  - if `marked == true` → clear `marked` for next cycle

This is the simplest “real” GC.

### Milestone C: Avoid recursion (important)
Marking should use an explicit stack/queue:
- recursion can blow the Java call stack on deep graphs

### Milestone D: Compaction or copying (optional next step)
#### Copying collector (semi-space)
- Maintain two spaces: `fromSpace` and `toSpace`
- Copy reachable objects into `toSpace`
- Update all references (roots + object fields)
- Swap spaces

#### Mark–compact
- Mark reachable
- Compute new addresses
- Move objects down to remove fragmentation
- Update references

### Milestone E: Generational GC (advanced)
- Allocate into a “young” space
- Minor GC copies survivors
- Promote to “old” after N survivals
- Add a write barrier + remembered set

---

## Starter pseudocode (mark–sweep)

### Mark (iterative)
- stack = []
- push all roots
- while stack not empty:
  - h = pop
  - if h invalid or heap[h] is null: continue
  - if heap[h].marked: continue
  - heap[h].marked = true
  - for each child in heap[h].refs:
    - push child

### Sweep
- for i in 0..heapSize-1:
  - obj = heap[i]
  - if obj == null: continue
  - if obj.marked == false:
    - heap[i] = null  // free
  - else:
    - obj.marked = false  // reset for next GC

---

## Suggested tests (make it feel real)

### Reachability
- Allocate a chain A → B → C, root A
  - run GC: all survive
  - remove root A, run GC: all collected

### Cycles
- A → B → A (cycle), root A
  - survives
- remove root
  - collected (proves reachability-based GC handles cycles)

### Stress
- Allocate many objects, keep only some reachable
- Run GC and ensure freed slots get reused

---

## Notes / gotchas you’ll learn naturally
- Root management matters (forgetting to register a root “loses” objects)
- Fragmentation motivates compact/copy collectors
- Generational needs write barriers to stay correct
- “Handles” make moving GC easier (you can update handle mappings)

---

## Next extensions (pick what’s fun)
- Add object “types” (like structs with named fields)
- Add a bytecode interpreter and GC its objects
- Add stats: allocated count, freed count, pause time
- Add incremental marking (hard but rewarding)
