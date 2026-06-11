# Clock-Aware Programming
#### Alex Chevrier — chevrier.alex@gmail.com

> One primitive — declared operation timing — propagates upward to eliminate an operating system, and downward to redesign silicon.

---

*The thinking in these papers is mine. The writing was produced with AI assistance (GitHub Copilot / Claude). I am not a professional researcher or academic — I am someone who followed an idea to its logical conclusion and wrote down what I found.*

---

**Started: 2026-06-07.**

---

## The Root Observation

Every complex mechanism in modern computing — the scheduler, locks, RCU, memory barriers, GC, interrupt dispatch, DMA coordination, cache coherency hardware, the branch predictor, the out-of-order engine — exists to compensate for the same missing fact: **when does this operation run, relative to every other operation that could conflict with it?**

Nobody declared the timing. Until they do, every layer must compensate at runtime for not knowing it at compile time. The compensation is not free. It compounds: no timing → locks → RCU → forced indirection → heap allocation → TLB pressure → cache misses → scheduling variance → more locks. Each layer adds overhead because the layer below could not provide a timing guarantee.

Clock-aware programming declares the timing. The compensation becomes unnecessary. What remains is the computation.

---

## Three Consequences

### I. Clock-Aware Programming

When every operation's timing is declared and compiler-verified, the runtime mechanisms that compensate for not knowing when things happen — schedulers, locks, RCU, memory barriers, GC — become provably unnecessary. Not deprecated. Not replaced by lighter versions. Provably redundant.

The same insight that makes FPGAs deterministic — the clock as global synchronisation primitive, timing constraints as compile-time certificates — applied to commodity CPUs under a narrow, achievable set of preconditions.

→ **[Paper I: Clock-Aware Programming](docs/papers/01-clock-aware-programming.md)**

The core primitive, the key claims, the implementation path (Rust + proc-macro + `system.cap`), and the hardest objection. This is the working paper — implementation decisions are in the ADRs.

---

### II. The Language

A language with four rules instead of four thousand pages — designed from the declarations down, not from a syntax up. No unsafe. No barriers. No move semantics. No GC. If it compiles, it is hardware-correct.

- **Four rules** — clock annotation, lifetime type, channel, exhaustive match. Complete. No fifth rule.
- **A function is a circuit** — its output is a return value (a signal), not a parameter. Composition is function application.
- **A class is a circuit with an ALU** — state is a register file, methods are combinational logic, timing is declared.
- **One guarantee** — if it compiles, it is hardware-correct. Not by convention. By construction.

→ **[Paper II: The Language](docs/papers/02-language.md)**

---

### III. The OS and Runtime

When the OS is written in the same language, the runtime and kernel converge. The scheduler, memory manager, driver model, permission system, and deployment model all dissolve into the same four rules. Running a program is adding a circuit. Booting is adding the OS circuit. Exceptions are signals on declared channels.

- **One runtime** — bitstream loader: loads the circuit manifest, pre-allocates memory, steps aside.
- **One kernel** — a collection of declared-timing circuits, indistinguishable from application circuits except by channel subscription.
- **Deterministic under any load** — the dispatch table is a theorem; runtime decisions are executions of that theorem.

→ **[Paper III: The OS and Runtime](docs/papers/03-os-and-runtime.md)**

---

### IV. Hardware Architecture Implications

A CPU designed for software that declares its schedule has no use for an out-of-order engine, branch predictor, or L2/L3 cache hierarchy. The freed die area — 70–90% of a modern CPU — is reinvested into execution ports, more cores, and clock rate. Unified memory removes the VRAM/DRAM boundary. The result is an architecture where every transistor performs computation, scaling better with every process node than any OOO processor.

- **One architecture** — unified memory, declared access windows, layered tiers, clock distribution as correctness primitive.
- **Not x86. Not ARM. Not RISC-V.** — defined by declarations, not instruction sets. Any silicon can implement it.
- **Hardware cost** — 3–4× reduction for equivalent inference throughput, because the transistors that remain all compute.

→ **[Paper IV: Hardware Architecture Implications](docs/papers/04-hardware-architecture.md)**

---

## The One Sentence

> "One architecture, one language, one runtime — fully hardware compliant, provably correct, and simple enough for anyone to use."

That is not a product pitch. It is the description of what computing looks like when you remove everything that exists only to compensate for undeclared timing. What's left is exactly what was always there underneath the complexity.

---

## Architecture Decision Records

The ADRs record implementation decisions within Paper I — the near-term path to a working prototype.

| # | Title | Status |
|---|---|---|
| [0001](docs/adr/0001-naming-clock-aware-programming.md) | Naming: clock-aware programming | Accepted |
| [0002](docs/adr/0002-cpu-partition-model.md) | CPU partition model (OS cores / app cores) | Accepted |
| [0003](docs/adr/0003-rcu-elimination-via-compile-time-proofs.md) | RCU elimination via compile-time scheduling proofs | Accepted |
| [0004](docs/adr/0004-rust-as-implementation-vehicle.md) | Rust as implementation vehicle | Accepted |
| [0005](docs/adr/0005-unified-system-configuration.md) | Unified system configuration file | Accepted |
| [0006](docs/adr/0006-poll-mode-self-regulating-network-stack.md) | Poll-mode self-regulating network stack | Superseded by 0010 |
| [0007](docs/adr/0007-memory-ordering-elimination.md) | Memory ordering elimination via compile-time scheduling proofs | Accepted |
| [0008](docs/adr/0008-clock-aware-memory-management.md) | Clock-aware memory management | Accepted |
| [0009](docs/adr/0009-implied-hardware-architecture.md) | The implied hardware architecture | Speculative |
| [0010](docs/adr/0010-channel-based-io.md) | Channel-based I/O — hardware signals as declared-timing channels | Accepted |
