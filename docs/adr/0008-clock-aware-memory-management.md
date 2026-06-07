# ADR-0008: Clock-Aware Memory Management

**Date:** 2026-06-07  
**Status:** Accepted

---

## Context

Garbage collection is the dominant memory management strategy in managed runtimes (JVM, .NET, Go, Python, JavaScript). Every GC design since the 1960s has been an attempt to solve the same problem: the collector does not know when the mutator (the running program) will next access a given object.

### Why GC is complex

Because the collector does not know mutator timing, it must either:

1. **Stop the world** — pause all mutator threads, collect, resume. Correct but produces stop-the-world pauses that scale with heap size.
2. **Run concurrently** — collect while the mutator runs, using write barriers and read barriers to track mutations the collector might have missed. Correct under careful implementation but adds barrier cost to every pointer write or read.

Every major GC innovation is a refinement of this tradeoff:

| GC design | Mechanism | Root cause it addresses |
|---|---|---|
| Generational | Young/old generation heuristic | Most objects die young — but "most" is a probabilistic claim, not a proof |
| Incremental | Break collection into small steps | Reduce pause length — but not eliminate pauses |
| Concurrent mark (CMS, G1) | Mark while mutator runs | Write barriers track mutations — but barriers add cost to every store |
| ZGC / Shenandoah | Colored pointers / load barriers | Read barriers on every pointer load — eliminates pauses at the cost of per-load overhead |
| Reference counting (Swift, Rust Arc) | Count holders at every assignment | No GC pause — but every retain/release is an atomic increment/decrement on the hot path |
| Region-based (Rust ownership) | Compiler proves lifetimes statically | No runtime cost — but requires programmer to declare ownership explicitly |

Rust's ownership model is the closest existing system to clock-aware memory management: it moves the proof from runtime to compile time. Its limitation is that it proves *ownership transfer* (which object owns this memory), not *temporal access* (when this memory is accessed). The two proofs are related but not identical.

### The compensating machinery

Every GC runtime includes:

- **Write barriers** — code injected at every pointer store to notify the collector.
- **Read barriers** — code injected at every pointer load (ZGC, Shenandoah) to remap relocated objects.
- **Card tables / remembered sets** — data structures tracking which old-generation regions contain pointers to young-generation objects.
- **Safepoints** — globally consistent states where all mutator threads can be paused; reaching a safepoint requires polling on every loop back-edge.
- **Finalizer queues** — deferred execution of cleanup code after an object is determined unreachable.
- **Concurrent marking threads** — background threads walking the object graph while the mutator runs.

Each of these exists entirely because the collector does not know when the mutator will access a given location. They are compensating machinery for a missing timing declaration.

---

## Decision

**Under clock-aware annotations, memory management reduces to a per-cycle two-state machine: allocate if requested, reclaim if the owning thread has completed its declared execution window.**

Concretely:

At every cycle boundary, two questions are asked — and only two:

1. **Did the thread request memory this cycle?** → allocate from the pre-declared partition.
2. **Did the thread complete its declared execution window this cycle?** → reclaim all memory allocated in that window.

Reclamation is immediate, at the boundary, with no grace period, no marking phase, no barrier — because the compiler has already proved at build time that no other thread holds a live reference to that memory past the declared window. The proof is a consequence of the timeslice annotation: if the owning thread's window is `[N, N+K]` and no other thread declares an access to that memory after cycle N+K, the memory is provably unreachable after the window closes.

Memory is not heap-allocated in the traditional sense. It is declared in a statically-known partition with a declared lifetime, the same way BRAM is allocated at FPGA synthesis time with a known address range and access schedule. The allocator does not search a free list; it advances a pointer within the declared partition. The collector does not trace a graph; it resets the partition pointer when the window closes.

---

## Rationale

### The GC problem is a timing problem

Every GC barrier, every write barrier, every remembered set, every safepoint poll exists because the collector does not know the answer to one question: is the mutator currently accessing this memory?

Clock-aware annotations answer that question at compile time. The mutator's access windows are declared. The collector's windows are declared. The compiler verifies they do not overlap. The question that drives all GC complexity has been answered before the program runs — and the complexity is unnecessary because the answer is already known.

### The two-question circuit

The per-cycle state machine is not a simplification of GC — it is the GC problem solved at its root:

```
per_cycle_boundary:
  if allocation_requested:
      ptr = partition.bump(size)   // O(1), no free list, no lock
  if execution_window_complete:
      partition.reset()            // O(1), no tracing, no marking
```

Every GC algorithm ever designed — generational hypothesis, tri-color marking, concurrent evacuation, colored pointers, safepoint polling — is a runtime approximation of this two-line circuit. The approximation is necessary when timing is unknown. When timing is declared, the approximation is replaced by the exact answer.

### Connection to Rust ownership

Rust's borrow checker proves memory safety at compile time for the ownership dimension: a value cannot be used after it has been moved, a mutable reference cannot coexist with other references. This eliminates use-after-free and data races without a GC.

Clock-aware memory management extends this to the temporal dimension: the declared execution window is a lifetime bound at the cycle level. `'timeslice(core=2, cycles=[N, N+K])` is a lifetime that expires at cycle N+K. The borrow checker already knows how to reason about lifetime expiry; the timeslice annotation provides the expiry point in hardware time rather than in program control flow.

The two systems compose: Rust ownership proves spatial safety (no aliased mutable references); clock-aware lifetimes prove temporal safety (no access after the declared window). Together they eliminate both GC and runtime synchronisation from the memory management model.

### Power and allocation efficiency

Traditional allocators (`malloc`, `kmalloc`, `new`) search a free list, potentially acquiring a lock, potentially triggering a page fault on first access. Under pressure, they call into the OS to extend the heap. Each of these operations has variable, unbounded latency.

A bump-pointer allocator within a declared partition has O(1) allocation latency, no lock (the partition is per-core by declaration), no page fault (the partition is pre-faulted at startup), and no OS involvement. Deallocation is a pointer reset — O(1), deterministic, immediate. The allocator is not a subsystem; it is two arithmetic instructions.

---

## Alternatives Rejected

### Generational GC with very short minor collections

Short minor GC pauses (< 1 ms) are acceptable for most workloads. They are not acceptable for hard real-time workloads (audio callbacks, modem frame processing, HFT order handling) where the deadline is 100 µs or less. Generational GC does not bound pause time; it reduces average pause time. The tail is unbounded.

### ZGC / Shenandoah (sub-millisecond concurrent GC)

ZGC achieves sub-millisecond pauses by adding load barriers to every pointer dereference. The barrier checks whether the pointer has been relocated and remaps it if so. This eliminates pauses at the cost of ~5–10% throughput overhead and a memory model that requires coloured pointer bits. The overhead is constant per load — it does not disappear under light load. For a system processing millions of operations per second, a constant per-load overhead is a structural cost, not a tuning parameter.

### Reference counting (Swift ARC, `Rc<T>`, `Arc<T>`)

Reference counting eliminates GC pauses by tracking object liveness at every assignment. The cost is an atomic increment/decrement on every clone/drop of a shared pointer — O(1) but present on every assignment, cache-hostile (the reference count is a separate cacheline from the data), and not compositional (cycles require weak pointers). For hot paths with frequent pointer copies, the atomic operations dominate the access cost.

### Stack allocation only

Stack allocation is O(1), has no GC overhead, and is automatically reclaimed at scope exit. Its limitation is size: kernel stack is 8 KB, user stack is typically 8 MB, and stack allocation cannot outlive the function that performed it. For data that must be shared across functions or threads, stack allocation is not available. Clock-aware partitions have neither the size constraint (the partition is declared at boot in a pre-allocated memory region) nor the scope constraint (the lifetime is a declared cycle window, not a function call depth).

### Rust ownership without timeslice extension

Rust's borrow checker proves memory safety without GC for a large class of programs. It cannot express "this memory is valid for cycles [N, N+K] on core 2" — the lifetime system is control-flow scoped, not time-scoped. Clock-aware memory management extends Rust's proof system into the temporal domain rather than replacing it.

---

## Consequences

- Memory for clock-aware code paths is allocated from a declared partition in `system.cap`. The partition has a declared address, size, and per-core layout.
- Allocation within the partition is bump-pointer: O(1), no lock, no OS call.
- Reclamation at cycle boundary is a pointer reset: O(1), immediate, no tracing.
- Objects whose lifetime spans multiple cycle windows must be declared explicitly — their window is the union of the windows that access them. The compiler verifies no access occurs outside the declared window.
- Objects shared across partitions (OS partition → app partition) are immutable after the OS partition's write window closes. The compiler enforces this via the `#[no_rcu]` proof (ADR-0003): if the write window is closed and the read window is open, the object is provably immutable during the read.
- The `system.cap` partition declaration is the authoritative memory map for clock-aware code. All allocation, reclamation, and cross-partition access is validated against it at compile time.
