# ADR-0003: RCU Elimination via Compile-Time Scheduling Proofs

**Date:** 2026-06-07  
**Status:** Accepted

---

## Context

Read-Copy-Update (RCU) is the Linux kernel's primary mechanism for allowing concurrent readers and writers without reader-side locks. It is used pervasively in kernel hot paths: the scheduler runqueue, the routing table, the socket hash table, the file descriptor table, and many others.

### Why RCU exists

Readers in the kernel must never block. If a reader acquires a lock and is preempted, the writer blocks, and kernel forward progress degrades. RCU solves this with deferred reclamation:

1. The writer atomically publishes a new pointer to updated data.
2. The old version remains valid until all *currently executing* readers have finished (the "grace period").
3. After the grace period, the old version is freed.

This guarantees that readers always see a consistent version without acquiring a lock. The cost:

- **Grace period tracking** — per-CPU state machines tracking quiescent states (context switch, idle, user-mode entry).
- **Callback queues** — deferred `kfree_rcu` callbacks processed after the grace period.
- **Memory barriers** — `rcu_dereference()` emits `READ_ONCE` + a data dependency barrier; `rcu_assign_pointer()` emits `smp_store_release`.
- **`rcu_nocbs` complexity** — on isolated cores (`nohz_full`), RCU callbacks must be offloaded explicitly; misconfiguration causes silent latency spikes.

### Why RCU forces indirection

RCU's publish-subscribe mechanism requires a single atomic word (pointer) to swap when updating. This forces heap allocation + pointer chasing even when a flat array would be the natural representation. The indirection is not a data structure choice — it is a mechanism requirement.

### The kernel memory constraint loop

This indirection compounds with the kernel's memory constraints:

- Kernel stack: 8 KB per thread (16 KB in some configurations). No guard page expansion.
- `kmalloc`: physically contiguous, size-limited, GFP flags add latency on the allocation path.
- `vmalloc`: virtually contiguous, TLB pressure, first-access page fault cost.

Because the stack is small: large flat structures cannot live there → forced to heap → pointer indirection → needs RCU → forces more indirection. This is a self-reinforcing loop.

### Hot-path datasets are small

The data protected by RCU in hot paths is not large:

- Scheduler runqueue: tens to low hundreds of tasks on a production server.
- Routing table hot path: fits in L2 cache on any modern CPU.
- File descriptor table: almost always sparse, small working set per process.
- Network socket hash: hot entries are a tiny fraction of the allocated table.

Synchronisation cost (RCU barriers, grace period tracking, callback processing) frequently dominates actual data access cost. The system pays P99.9-level overhead to protect P50-level data volumes.

---

## Decision

**Under the clock-aware programming model, RCU is eliminated. Not bypassed, not reduced — eliminated. The premise of RCU is that reader timing is unknown at compile time; clock-aware annotations remove that unknowing. When every access to shared data is declared with a core and a cycle, the compiler proves ordering statically for all of them. The runtime mechanism exists to enforce ordering under temporal uncertainty — remove the uncertainty and the mechanism is unnecessary.**

Concretely:

- A `#[no_rcu]` attribute (Rust) or equivalent annotation is valid only when a compile-time checker can prove that the annotated read and any concurrent write occur in non-overlapping cycle windows on their respective cores.
- The checker operates on the partition model (ADR-0002) and system configuration (ADR-0005): it knows which core reads, which core writes, at what declared cycle each access occurs, and the latency of the read operation under the declared microarchitecture.
- If the proof passes, the runtime RCU machinery — grace period tracking, memory barriers, callback queues — is not emitted for that code path.
- Adopted progressively across the kernel, every annotated code path replaces a runtime RCU proof with a compile-time one. The RCU call sites that remain are the ones not yet annotated, not the ones that cannot be.

---

## Rationale

### The invariant RCU enforces can be stated as a compile-time predicate

RCU guarantees: a reader that begins before a writer publishes a new version will complete before the old version is freed. In temporal terms: `read_start(N) + read_latency < write_publish(M)` or `read_start(N) > write_publish(M)`. One of these must hold.

In a clock-aware system: `read_start` is a declared cycle on a declared core. `write_publish` is a declared cycle on a declared core. `read_latency` is computable from the instruction mix and the microarchitecture model. The predicate is fully determined at compile time.

This is the same invariant. The difference is where it is verified: runtime grace period vs compile-time proof.

### Removing RCU removes forced indirection

Without RCU, data does not need to be heap-allocated and pointer-referenced to support atomic pointer swap. Flat arrays — cache-line aligned, prefetchable, without pointer chasing — become legal representations for data that was previously forced into heap-allocated linked structures.

The combined effect:

1. Compiler proves access ordering → no RCU.
2. No RCU → no forced pointer indirection.
3. No pointer indirection → flat, cache-line-aligned structures.
4. Flat structures → prefetchable, no TLB pressure.
5. Small hot-path dataset → fits entirely in L2 or L1.

The kernel gets leaner at every layer simultaneously. The synchronisation layer, the data structure layer, and the memory access layer all improve together because their costs were coupled through the RCU mechanism.

### Breaking the memory constraint loop

If access schedule is known at compile time, working memory can be allocated once at boot in a declared partition — not on the kernel stack, not via `kmalloc`, not via `vmalloc`. The 8 KB stack limit becomes irrelevant. Memory layout is a compile-time declaration, not a runtime decision — the exact analogy to BRAM allocated at FPGA synthesis time.

### Framing for the kernel community

The proposal is not "remove RCU from the kernel tree in one patch" — that requires annotating every call site simultaneously, which is impractical. The proposal is: **for timeslice-annotated code paths, the type system can prove RCU is unnecessary and elide it.** Opt-in, backward compatible, demonstrably correct. As annotation coverage grows, RCU coverage shrinks. The `rust-for-linux` team is actively seeking cases where Rust's type system can eliminate kernel synchronisation primitives. This is that case.

---

## Alternatives Rejected

### PREEMPT\_RT + priority inheritance

Makes locking latency more predictable but does not eliminate locks. RCU grace periods still exist. The synchronisation cost is reduced, not removed.

### Seqlock

A seqlock allows a writer to proceed without waiting for readers by using a sequence counter. Readers retry if they observe an inconsistent sequence. This reduces writer blocking but adds reader retry cost and does not produce a compile-time proof. Under high write frequency, reader retry rate becomes unbounded.

### Hazard pointers

Hazard pointers are an alternative deferred reclamation scheme with lower grace period cost than RCU in some workloads. They still require runtime pointer registration and scanning, and they still force pointer indirection for the same structural reasons as RCU. The problem is not the grace period mechanism — it is the runtime nature of the ordering proof.

### Userspace RCU (liburcu)

Moves the RCU mechanism to userspace. Still a runtime proof. Still forces pointer indirection. The problem is not which side of the kernel boundary the mechanism runs on — it is that the ordering proof happens at runtime rather than compile time.

---

## Consequences

- `#[no_rcu]` (or equivalent) is valid on any code path where the access core and cycle are declared and the checker passes.
- The compiler must have access to `system.cap` (ADR-0005) at compile time to know core assignments, cycle budgets, and microarchitecture model.
- The first concrete implementation is a proc-macro crate in Rust that performs the proof at compile time and emits a compiler error if the cycle windows overlap.
- A kernel patch replacing one RCU read-side critical section with `#[no_rcu]` is the intended proof-of-concept upstream contribution, targeting `rust-for-linux`.
- Full elimination of RCU from a codebase is the end state of complete annotation coverage. Partial adoption is valid: unannotated code paths continue to use RCU normally.
- Memory barrier elimination follows the same proof structure and is addressed in ADR-0007. Memory management elimination follows in ADR-0008. All three — RCU, barriers, GC — are consequences of the same missing primitive.
