# ADR-0007: Memory Ordering Elimination via Compile-Time Scheduling Proofs

**Date:** 2026-06-07  
**Status:** Accepted

---

## Context

Modern CPUs and compilers reorder instructions. The CPU's out-of-order execution engine reorders at the microarchitectural level; the compiler reorders at the IR level to improve instruction scheduling and register allocation. Both are correct transformations in isolation — they preserve the observable behaviour of a single-threaded program — but they produce incorrect behaviour when two cores access shared data and one core assumes the other's writes are visible in program order.

The consequence: every cross-core shared memory access in a multi-threaded program must be annotated with an ordering constraint, or the access is data-race undefined behaviour.

### The cost of runtime memory ordering

The mechanisms are expensive:

- **`smp_mb()` / full barrier** — serialises the store buffer and load buffer on both sides. On x86 this is `MFENCE` or a locked instruction; on ARM it is `DMB ISH`. Cost: tens of cycles, prevents all reordering across the barrier.
- **`acquire` / `release`** — one-sided barriers. `acquire` prevents loads from being moved before the barrier; `release` prevents stores from being moved after it. Cheaper than a full barrier but still serialises part of the pipeline.
- **`READ_ONCE` / `WRITE_ONCE`** — prevent the *compiler* from caching or tearing the access; do not prevent CPU reordering on weakly-ordered architectures (ARM, POWER, RISC-V).
- **`rcu_dereference()`** — `READ_ONCE` + an implicit data dependency barrier on architectures where pointer-chased loads are not ordered by the dependency.
- **`std::atomic` with `memory_order_seq_cst`** — full sequential consistency; most expensive.

In the Linux kernel, these appear on every hot path. The scheduler, the network stack, the VFS, RCU itself — every subsystem inserts barriers because no subsystem knows the timing of any other.

### Why barriers exist: the same missing fact

A memory barrier is a runtime assertion of the form: "whatever was written before this barrier must be visible to any core that executes a load after its corresponding barrier." It is inserted because the programmer knows that *some* ordering must hold but cannot prove *which* ordering the hardware will produce without the barrier.

The underlying cause is identical to RCU (ADR-0003) and the scheduler (ADR-0002): the system does not know when accesses happen relative to each other, so it must enforce ordering at runtime rather than proving it at compile time.

---

## Decision

**Under clock-aware annotations, memory barriers are eliminated for any pair of accesses whose relative order is provable from their declared cycle windows.**

Concretely:

- If core A writes at declared cycle N and core B reads the same location at declared cycle M where M > N + write_propagation_latency, the write is provably visible to the read — by the schedule, not by a barrier.
- The compile-time checker (the same proc-macro crate from ADR-0004) verifies this relationship. If it holds, no barrier is emitted for that access pair.
- If the relationship cannot be proved statically (cycles not declared, or windows overlap), the checker rejects the `#[no_barrier]` annotation and requires an explicit barrier — preserving correctness.

This is the same proof structure as RCU elimination (ADR-0003): a runtime enforcement mechanism becomes unnecessary when the invariant it enforces can be derived at compile time from declared scheduling information.

---

## Rationale

### Memory ordering is a timing problem, not a hardware problem

Weak memory models (ARM, POWER, RISC-V) are often framed as a hardware complication. They are not — they are an accurate description of what the hardware does when it is not constrained. The CPU reorders because reordering is faster and no one told it not to. The barrier tells it not to, at runtime, at the specific point where ordering matters.

Clock-aware annotations tell the compiler the ordering at compile time. On a weakly-ordered architecture, if the compiler knows that core A's store at cycle N is guaranteed to have propagated before core B's load at cycle M, it can emit the access without a barrier and the ordering is guaranteed by the schedule — not because the CPU was told to slow down, but because the timing window makes out-of-order visibility impossible within the declared parameters.

This is exactly how FPGA timing constraints work. An XDC `set_max_delay` constraint does not add a barrier between two registers — it proves that the signal propagation time is bounded and then relies on that bound. The hardware does what it does; the constraint proves it is correct.

### The scope of elimination

The barrier cost in real systems is not marginal:

- The Linux kernel contains thousands of `smp_mb()`, `smp_rmb()`, `smp_wmb()` calls on hot paths.
- Every `rcu_dereference()` on a non-x86 architecture includes an implicit barrier.
- Every `std::atomic::load(memory_order_acquire)` in a userspace lock-free structure includes a barrier.
- Every `pthread_mutex_lock` includes acquire/release semantics.

Under full clock-aware annotation coverage, all of these become unnecessary for the annotated code paths. The CPU runs without serialisation at those points — not unsafely, but correctly by construction, because the timing proof guarantees the ordering.

### Architecture-awareness: barriers are architecture-specific, proofs are not

A full barrier on x86 costs ~20–40 cycles. The same barrier on ARM can cost more due to the weaker memory model requiring more aggressive flushing. A program that relies on barriers has architecture-dependent performance that must be re-tuned per target.

A timing proof is architecture-independent in structure but architecture-specific in parameters: the write propagation latency term is a property of the `cpu_model` declared in `system.cap`. The same proof runs on x86 and ARM; it uses different latency values. The program is portable; the timing parameters are derived, not assumed.

### Connection to the simplification thesis

Memory barriers are the third major category of compensating machinery — alongside RCU (ADR-0003) and the scheduler (ADR-0002) — that exists entirely because timing is not declared. Removing the declaration gap removes all three categories simultaneously, not sequentially. They are not three separate problems; they are three symptoms of the same missing primitive.

---

## Alternatives Rejected

### Stronger hardware memory models

x86's Total Store Order (TSO) model eliminates most load-load and store-store reordering, making barriers cheaper and rarer than on ARM. This is not a solution — it is a mitigation. TSO still requires `MFENCE` for store-load reordering, and it only applies to x86. ARM, RISC-V, and POWER systems (including all mobile SoCs) remain weakly ordered and require explicit barriers. A solution that works on x86 and requires re-engineering on ARM is not a solution; it is a hardware dependency.

### Transactional memory (HTM)

Intel TSX and equivalent transactional memory extensions allow a region of code to execute optimistically and roll back on conflict. This eliminates locks in the common case but does not eliminate memory ordering: the commit of a transaction still requires ordering relative to other transactions. Transactional memory addresses lock contention; it does not address the barrier problem.

### Sequential consistency by default

Defaulting all atomics to `memory_order_seq_cst` (Java's model, early C++ default) eliminates programmer error at the cost of maximum barrier insertion on every atomic operation. This makes the barrier problem worse, not better. It is the safe choice for correctness; it is the wrong choice for performance.

### Static analysis of existing code (without annotations)

Static race detectors (ThreadSanitizer, Helgrind) can identify missing barriers in existing code. They cannot prove that a barrier is *unnecessary* — they can only detect that one is missing. Proving absence of a race requires knowing the ordering, which requires knowing the timing, which requires annotations. Static analysis without annotations cannot replace the annotations.

---

## Consequences

- `#[no_barrier]` (or equivalent) is valid only when the compile-time checker can prove the ordering from declared cycle windows.
- The checker uses the `cpu_model` from `system.cap` to derive write propagation latency for the target architecture. Correctness of the proof depends on the accuracy of this model.
- On weakly-ordered architectures (ARM, RISC-V), the elimination is more significant — more barriers are required today, and more are provably unnecessary under clock-aware annotations.
- Code paths that mix annotated and unannotated accesses to the same memory location require explicit barriers at the annotated/unannotated boundary. The checker enforces this.
- The full elimination of barriers from a codebase is the end state of complete annotation coverage. Partial adoption eliminates barriers only for the annotated paths; unannotated paths retain their existing barriers.
