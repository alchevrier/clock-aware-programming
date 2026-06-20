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

## Implementation Roadmap

The prototype target is a **Raspberry Pi 3B+** — Cortex-A53 (ARMv8-A), 4 cores at 1.4GHz, bare metal. No Linux. No FPGA. The model runs on commodity hardware that anyone can buy for $35.

**Phase 0 — Compiler → manifest**
Parse the core keywords, walk the instruction graph, derive `budget_ticks` per circuit against the Cortex-A53 instruction latency table (ARM Software Optimization Guide). Run the admission test: `Σ budget_ticks ≤ epoch_cycles` per core. Emit a manifest. The feed handler circuit compiles and produces a valid manifest. Target architecture: `aarch64-unknown-none`.

**Phase 1 — Runtime on Pi 3B+, bare metal**
Manifest loader. Dispatch table per core. Channel regions pre-allocated in SDRAM from the manifest. Four circuits, nothing more:

- `StorageCircuit` — reads a NASDAQ ITCH data file from SD card, writes one record per window to `channel NasdaqRecord`
- `FeedHandlerCircuit` — reads `channel NasdaqRecord`, processes each record (parse, book update), writes result to `channel BookSnapshot`
- `ClockCircuit` — wraps `CNTPCT_EL0`, timestamps each circuit window entry and exit
- `ObservabilityCircuit` — reads timestamps, accumulates per-record cycle counts, computes p50 / p99 / p99.9 distribution, writes results to `channel DisplayLine`
- `DisplayCircuit` — owns the HDMI framebuffer as a `permanent` channel, renders `channel DisplayLine` writes as text pixels

At end of file: the percentile results are written to `channel DisplayLine`. `DisplayCircuit` owns the HDMI framebuffer (allocated via the VideoCore mailbox at boot, mapped as a `permanent` channel) and renders the values directly to screen. Plug in HDMI, power on, see results. No serial cable. No second machine. No terminal emulator. **The question the benchmark answers: did the model predict the latency correctly?**

**Why the numbers will be competitive against optimised C++ on an i5.** The comparison is not raw throughput — a 1.2GHz in-order Cortex-A53 does not race a 4GHz OOO i5. The comparison is tail latency and the predictability of p50. Two structural advantages the model owns that no amount of C++ optimisation can recover:

- **p99.9 — OS jitter eliminated by construction.** Linux on an i5, even with `SCHED_FIFO` and CPU isolation, has irreducible jitter: timer interrupts, TLB shootdowns, kernel preemption points, OOO pipeline variance. These appear as unpredictable spikes in p99.9 that the application cannot explain and cannot eliminate. Clock-aware bare metal has none of them. The dispatch table is the only thing that runs. The p99.9 the compiler printed before the program ran is the p99.9 the hardware delivers.

- **p50 — speculative pre-loading is exact, not heuristic.** On the i5, the hardware prefetcher observes access patterns and guesses what to pre-load into L1. For a regular feed handler pattern it is often right — but "often" leaves a residual miss rate that shows up in the p50 distribution. On the clock-aware runtime, the dispatch table is known before execution begins. The runtime knows exactly which channels `FeedHandlerCircuit` reads, their memory tier, and when the circuit opens. During the preceding circuit's window, the runtime issues `PRFM` (ARM software prefetch) for exactly the cold addresses `FeedHandlerCircuit` will need — not a guess, a proof derived from the manifest. By the time the window opens, the data is in L1. Zero cache miss cost, not reduced miss cost. On the Cortex-A53's in-order pipeline, `PRFM` timing is deterministic; on the i5's OOO engine, software prefetch timing is not, because instruction ordering is dynamic.

Nobody arrives at "zero cache misses by construction" from the hardware optimisation side. Every hardware tool — prefetcher hints, `CLFLUSH`, huge pages, NUMA affinity — is a local optimisation applied after the fact. The ceiling of those optimisations is a reduced miss rate. The model's ceiling is a proof. The difference in outcome is not a matter of trying harder. It is a matter of starting from a different question.

**Phase 2 — Kernel circuits promoted**
`ClockCircuit`, `StorageCircuit`, and `MemoryCircuit` become declared kernel circuits running on the same dispatch table as the application circuits, indistinguishable by the runtime. The swap protocol demonstrated live: replace `StorageCircuit` without stopping execution.
