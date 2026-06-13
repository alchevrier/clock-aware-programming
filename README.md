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

## Four Consequences

### I. Clock-Aware Programming

When every operation's timing is declared and compiler-verified, the runtime mechanisms that compensate for not knowing when things happen — schedulers, locks, RCU, memory barriers, GC — become provably unnecessary. Not deprecated. Not replaced by lighter versions. Provably redundant.

The same insight that makes FPGAs deterministic — the clock as global synchronisation primitive, timing constraints as compile-time certificates — applied to commodity CPUs under a narrow, achievable set of preconditions.

→ **[Paper I: Clock-Aware Programming](docs/papers/01-clock-aware-programming.md)**

The core primitive, the key claims, the implementation path (Rust + proc-macro + `system.cap`), and the hardest objection. Covers: CPU partition model, channel subscriptions, the FPGA parallel, irregular workloads, and why every production system already declares the bounds required.

---

### II. The Language

A language with four rules instead of four thousand pages — designed from the declarations down, not from a syntax up. Twelve reserved lowercase keywords. No unsafe. No barriers. No move semantics. No GC. No generics. No type parameters. If it compiles, it is hardware-correct.

- **Four rules** — clock annotation, lifetime type, channel, exhaustive match. Complete. No fifth rule.
- **Twelve keywords** — `fn`, `circuit`, `channel`, `clock`, `val`, `register`, `ephemeral`, `task`, `session`, `permanent`, `memory`, `cold`. Every construct in the language is built from these.
- **`fn` is combinational logic** — output is a return value (a signal), not a parameter. Composition is function application.
- **`circuit` is a clocked block** — internal state declared as self-feeding `channel` entries. The flip-flop model, not OOP.
- **`channel` is the only data primitive** — typed, sized, tiered, with declared access patterns. Ring buffers, queues, stacks, lookup tables: all `channel` declarations. No separate collection model.
- **Compiler derives all costs** — `element` + `tier` + `size` on every channel declaration gives the compiler exact `put`/`get` cycle cost from first principles. No estimates, no budget tables, no profiling required.
- **Designed for AI authorship** — four rules fit in any context window. The compiler is the sole correctness gatekeeper. No human code review required for safety-critical properties.

→ **[Paper II: The Language](docs/papers/02-language.md)**

---

### III. The OS and Runtime

When the OS is written in the same language, the runtime and kernel converge. The scheduler, memory manager, driver model, permission system, filesystem, and deployment model all dissolve into the same four rules. Running a program is adding a circuit. Booting is adding the OS circuit. Exceptions are signals on declared channels.

- **Bitstream loader** — the runtime loads the circuit manifest, pre-allocates all memory from a static resource profile, and steps aside. No dynamic allocator.
- **Kernel is circuits** — every OS service is a declared-timing circuit, indistinguishable from application circuits except by channel subscription. No privilege rings needed.
- **Execution is a FSM** — IDLE → PLAN → EXECUTE → EVALUATE. The dispatch table is a compile-time theorem; runtime decisions are executions of that theorem.
- **Register forwarding and multi-stack pipeline** — the hardware SP is reserved for the runtime only; circuits use software-managed register stacks.
- **Memory manager is two questions** — which tier, which circuit. All allocations are compile-time decisions; `out of memory` is a circuit removal event.
- **ARM hardware mapped explicitly** — clock domains (CLKIN/CNTCLKEN/ATCLKEN), ACP for accelerator coherency, core power modes (STANDBYWFI2/Retention/Dormant), gathering/non-gathering memory regions.
- **Cryptographic circuit identity** — circuits are signed at compile time; the runtime verifies signatures before execution. Privilege rings, Spectre/Meltdown mitigations, SMEP/SMAP, ASLR become structurally irrelevant.
- **AI-regulated OS** — the runtime adapts clock frequency, L1 pre-population, and core affinity using the compile-time dispatch table as lookahead. Speculative pre-conditioning, not reactive compensation.
- **Native substrate for ML** — declared memory tiers, compile-time weight placement, channel-based tensor routing, GPU as mathematical circuit array.

→ **[Paper III: The OS and Runtime](docs/papers/03-os-and-runtime/index.md)**

---

### IV. Hardware Architecture Implications

A CPU designed for software that declares its schedule has no use for an out-of-order engine, branch predictor, or L2/L3 cache hierarchy. The freed die area — 70–90% of a modern CPU — is reinvested into execution ports, more cores, and clock rate. Unified memory removes the VRAM/DRAM boundary. The result is an architecture where every transistor performs computation, scaling better with every process node than any OOO processor.

- **Unified memory** — DRAM and on-device memory on a single bus; declared access windows replace coherency hardware.
- **Layered tiers** — `register`, `ephemeral`, `task`, `session`, `permanent`, `cold` map directly to silicon: register file, L1, L1/L2, DRAM, pinned DRAM, NVMe.
- **Die area reallocation** — OOO engine, branch predictor, rename tables, speculation buffers removed; area reinvested in wide execution ports, large register file, more cores.
- **Process scaling advantage** — without speculation machinery, each new process node translates directly into more compute, not more speculative complexity.
- **GPU as circuit array** — channels are broadcast; one declared signal reaches all subscribed accelerators. Precise and imprecise resources unified under the same declaration model.
- **New RAM technologies** — HBM, LPDDR, NVMe-over-fabric all expressible as declared tiers with latency bounds in `system.cap`.
- **Not x86. Not ARM. Not RISC-V.** — defined by declarations, not instruction sets. Any silicon can implement it. Hardware cost: 3–4× reduction for equivalent inference throughput.

→ **[Paper IV: Hardware Architecture Implications](docs/papers/04-hardware-architecture.md)**

---

## The One Sentence

> "One architecture, one language, one runtime — fully hardware compliant, provably correct, and simple enough for anyone to use."

That is not a product pitch. It is the description of what computing looks like when you remove everything that exists only to compensate for undeclared timing. What's left is exactly what was always there underneath the complexity.

---

## Architecture Decision Records

The ADRs record implementation decisions for the near-term Rust prototype (`Channel<T>` syntax, proc-macro annotations, `system.cap`, `cargo-timeslice`). The prototype is a research vehicle — a way to validate the four rules on existing hardware. The language described in the papers is a separate, later destination.

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
