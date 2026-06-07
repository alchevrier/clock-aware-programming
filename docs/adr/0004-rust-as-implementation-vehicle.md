# ADR-0004: Rust as Implementation Vehicle

**Date:** 2026-06-07  
**Status:** Accepted

---

## Context

Clock-aware programming requires a language that can:

1. Express timing constraints as verifiable annotations, not comments.
2. Enforce the constraint at compile time, not at runtime.
3. Integrate with the Linux kernel (for the RCU elimination contribution in ADR-0003).
4. Support low-level, zero-overhead, `unsafe` operations where hardware interaction is required.

The choice of language determines what the annotations look like, what the compiler can prove, and whether the contribution has a viable path to kernel mainline.

---

## Decision

**Rust is the implementation vehicle for clock-aware annotations, the compile-time checker, and the kernel contribution.**

Specifically:

- Clock-aware annotations are expressed as Rust attributes: `#[timeslice(core = 2, cycle = N, budget_ns = 4)]`.
- The compile-time checker is a proc-macro crate that verifies annotated accesses do not produce overlapping cycle windows.
- The `#[no_rcu]` attribute is valid only when the checker passes.
- The kernel contribution targets `rust-for-linux`, where Rust is already an accepted language with Linus Torvalds' explicit support.

---

## Rationale

### The borrow checker already proves the invariant RCU enforces

Rust's borrow checker proves read/write exclusivity at compile time: a mutable reference (`&mut T`) cannot coexist with any other reference to the same data. This is structurally the same guarantee RCU provides at runtime — a reader will not observe a partially-written value.

Adding timeslice annotations extends this existing proof system. The borrow checker proves *spatial* exclusivity (no aliased mutable references); the timeslice checker proves *temporal* exclusivity (no overlapping cycle windows). The combination eliminates the need for the runtime RCU mechanism entirely.

Building on an existing, sound proof system is feasible. Building a new one from scratch in C is not — C has no ownership model for the type system to extend.

### Lifetimes are a natural hook for timeslice semantics

Rust lifetimes encode validity ranges for references. A timeslice annotation fits naturally as a lifetime bound:

```rust
fn read_order_book<'timeslice(core=2, cycle=N)>(
    book: &'timeslice OrderBook
) -> BestBid { ... }
```

The annotation reads: "this reference is valid during cycle N on core 2." The type system can reason about whether two such references with different cycle annotations can coexist — which is exactly the proof needed for RCU elimination.

### `unsafe` boundary already exists for hardware-level operations

Clock-aware code that directly accesses hardware-mapped memory (NIC ring buffers, MMIO registers) belongs in `unsafe` blocks. Rust's `unsafe` boundary is precisely scoped: the compiler verifies everything outside `unsafe`; the programmer takes responsibility for invariants inside it. This is the right model for hardware interface code — it does not pretend safety where it cannot be proven, and it does not sacrifice the proof for the rest of the codebase.

Clock-aware annotations that govern hardware access sit at this boundary. The annotation is the declaration of the invariant; the `unsafe` block is where the hardware access happens; the checker verifies the annotation is satisfied before permitting the `unsafe` code.

### `rust-for-linux` is a real community path

C is the established language of the kernel. A C extension or C preprocessor macro scheme for clock-aware annotations would need to be proposed as a new kernel subsystem — a high bar with no existing constituency.

Rust-for-Linux is different:

- Linus Torvalds has explicitly accepted Rust as a second implementation language for kernel drivers and subsystems.
- The `rust-for-linux` maintainers are actively soliciting cases where Rust's type system eliminates kernel synchronisation primitives.
- A proc-macro crate implementing the timeslice checker can be developed, tested, and reviewed entirely outside the kernel tree, then presented to the kernel community with a working proof-of-concept patch.

The contribution path is: proc-macro crate (standalone) → kernel patch replacing one RCU critical section → RFC to `rust-for-linux` mailing list. This is a real sequence with known actors, not a theoretical path.

---

## Alternatives Rejected

### C with preprocessor macros

C has no compile-time type system extension mechanism beyond macros and `_Generic`. A clock-aware annotation in C would be a macro that expands to a runtime assertion at best, or a comment at worst. Neither provides a compile-time proof. The kernel is C; a C implementation has zero friction for upstreaming, but it cannot deliver the core guarantee (compile-time verification) that makes the model useful.

### C++ attributes (`[[timeslice(...)]]`)

C++ attributes can trigger compiler plugin behaviour. `llvm-mca` already consumes C++ source to perform static throughput analysis. A Clang plugin that processes `[[timeslice(...)]]` is technically feasible. The problem: the Linux kernel does not use C++ and is not open to C++ contributions. The plugin approach has no path to kernel integration and would require a separate ABI boundary. Useful for userspace; irrelevant for the kernel contribution.

### A new language

A purpose-built language with clock-aware semantics as a first-class primitive would be the ideal end state. It is not the right starting point: no compiler, no toolchain, no community, no kernel integration path. It would take the concept from "publishable whitepaper" to "years of compiler infrastructure work" with no guarantee of adoption. The decision is not about what the ideal tool would be; it is about what can deliver a credible, verifiable, kernel-adjacent implementation in the near term.

### Zig

Zig has excellent comptime evaluation and zero-overhead `unsafe`-equivalent semantics. It does not have a Rust-equivalent ownership model, so the proof of read/write exclusivity that Rust's borrow checker provides does not exist. More importantly, Zig has no kernel integration path comparable to `rust-for-linux`. The comptime evaluation is powerful but would require building the cycle proof system from scratch rather than extending an existing sound type system.

---

## Consequences

- The proc-macro crate is the first concrete deliverable. It must expose `#[timeslice(...)]` and `#[no_rcu]` attributes and emit compile errors on violation.
- The crate requires access to the system configuration (ADR-0005) at compile time to know core assignments and cycle budgets.
- The kernel contribution is a Rust patch, not a C patch, and targets `rust-for-linux` drivers or subsystem code — not the core RCU implementation.
- If `rust-for-linux` grows to cover scheduler or networking subsystems before this contribution is ready, the target code path for the proof-of-concept patch should be updated accordingly.
