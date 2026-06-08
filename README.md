# Clock-Aware Programming

> Static scheduling as a first-class language primitive: clock-aware annotations that let the compiler prove synchronisation unnecessary.

**Started: 2026-06-07.** This repository is a first-principles record in ADR form, updated as understanding deepens.

---

## Unified Vision

The end state is a system that behaves as a **self-regulating circuit with known timing**. Not an approximation of one, not a circuit emulated in software with runtime guards — an actual circuit, in the sense that every operation's timing is declared, compiler-verified, and architecture-aware. When load changes, the system adjusts its own poll rate, cycle budgets, and configuration parameters at the next clock boundary, from within, without OS intervention. When the binary is compiled for a different microarchitecture, the declared timings recompute automatically from the new `cpu_model` in `system.cap` — the same program, correct on Skylake and on Zen 4, because the timing is derived from the architecture rather than hardcoded against it.

This is what FPGAs do. Timing constraints in an XDC file are not constants — they are targets the toolchain meets for the specific silicon and temperature corner it is targeting. Clock-aware programming brings the same property to software: timing declarations that are architecture-aware, statically verified, and self-consistent by construction.

---

## The Core Insight

FPGAs are deterministic because the clock is the global synchronisation primitive. When a signal crosses a clock domain boundary, you declare that crossing and the toolchain proves it is safe. No runtime locks. No runtime races. Correctness is a compile-time certificate.

But the deeper insight is this: **when you build an FPGA system, you are building a CPU architecture.** You define the pipeline stages, the data paths, the state machines, the memory layout, the timing between every operation. You are not programming a processor — you are designing the hardware that computes. The distinction between "software" and "hardware" dissolves: your HLS description *is* the architecture, and the toolchain synthesises it into silicon.

Clock-aware programming applies this realisation to commodity CPUs. The CPU already exists — but when you declare every operation's timing, every core's role, every data path's access cycle, you are effectively designing the architecture above the silicon. The compiler becomes the synthesis tool. The annotations become the constraint file. The result is a program that does not run *on* an architecture so much as *define* one in software, portable across microarchitectures because the timing is derived from the target rather than hardcoded against it.

### The paradox of modern programming

Modern software engineering has spent decades trying to *remove* the sequential and temporal nature of computation. Async/await hides when code runs. Garbage collection hides when memory is released. Event loops hide ordering. Futures and promises decouple cause from effect in time. Thread pools hide which core executes which work. The stated goal is always abstraction: free the programmer from thinking about timing and sequence so they can focus on logic.

But timing and sequence are not incidental properties of computation — they are the properties a CPU is built from. Every instruction has a position in a sequence. Every operation has a cycle count. Every memory access has a latency relative to the access before it. The CPU does not execute logic; it executes a sequence of operations in time. That is the only thing it does.

When programming abstractions hide timing and sequence, they do not make the timing and sequence go away — they make it *uncontrolled*. The CPU still executes in strict sequence, still runs in real time, still pays cache miss latency and pipeline stall costs. It does so invisibly, outside the model the programmer is reasoning in. The result is a system that is correct at the logical level and unpredictable at the physical level — which is why performance engineering is empirical: profilers, latency histograms, and tuning knobs exist to recover the timing information that the programming model discarded.

Clock-aware programming inverts this. Timing and sequence are not hidden — they are declared. The programmer works *with* the CPU's fundamental nature rather than building an abstraction layer that fights it. The result is not a more constrained programming model; it is a more honest one.

CPUs are not deterministic — but only because we never tell them to be. Under a narrow, achievable set of preconditions, a CPU core can be made to behave like FPGA fabric:

| Precondition | Mechanism |
|---|---|
| Working set declared and pinned in L1 (< 32 KB) | `mlock`, huge pages, prefetch annotations |
| Dedicated isolated core | `isolcpus`, `nohz_full`, `irqaffinity`, `rcu_nocbs` |
| Branch-free hot path | Compiler intrinsics, profile-guided branch elimination |
| Poll-mode I/O | DPDK/XDP, no interrupt delivery to the hot core |

Under these preconditions: **cycle count is a pure function of instruction mix × CPU microarchitecture model — computable at compile time.**

That makes the clock a first-class compiler concept. A program that declares its timing constraints can have them verified statically, the same way a program that declares its type constraints has them verified statically.

---

## Key Claims

1. **Deterministic at compile time** — `llvm-mca` already performs static throughput analysis; clock-aware annotations (`[[timeslice(4ns)]]`) surface that analysis as a verifiable constraint visible in the IDE, not a post-hoc profiler output.

2. **Lock-free by construction** — Locks exist because of temporal uncertainty. Remove the uncertainty and ordering is guaranteed by schedule, not runtime serialisation. A compiler that knows core 2 reads at cycle N and core 0 writes at cycle N+K where K > read latency has proved the ordering. No lock needed.

3. **FSM execution model** — The app partition is a finite state machine: declared states, deterministic transitions, no preemption. The OS partition is a black box that produces inputs at declared rates via a ring buffer boundary. This is exactly the PS/PL split on a Zynq — except it runs on a commodity server.

4. **Unified system configuration** — Today, isolating a core requires `isolcpus=2-7 nohz_full=2-7 irqaffinity=0,1 rcu_nocbs=2-7` scattered across boot parameters, systemd unit files, and NUMA policy scripts. One declarative config file replaces all of it and scopes the compiler's verification domain.

5. **Self-regulating network stack** — Eliminates the `hardirq → softirq → sk_buff → socket` chain. The NIC is a memory-mapped ring buffer polled at a declared cycle interval. Poll rate self-regulates on ring depth: deep ring → poll faster; empty ring → back off. No kernel involvement on the hot path.

6. **The scheduler becomes unnecessary** — The scheduler exists because the OS does not know when tasks will complete; it arbitrates contention between them by preemption. But the scheduler can only arbitrate *contention* — and contention only exists between circuits that share time without declaring it. Under clock-aware programming, every circuit in the kernel is independent and self-regulating by declared clock cycles: each occupies non-overlapping cycle windows, self-regulates on load within its own declared budget, and does not compete with any other circuit for CPU time because competition requires temporal ambiguity. Independent circuits with declared, non-overlapping windows have no contention to arbitrate. This applies not just to user processes but to the kernel itself: softirqs, RCU callbacks, kthreads, and workqueues are all scheduled because their timing is unknown and they implicitly contend. Make every circuit's timing explicit and the contention dissolves by construction — not because the scheduler became smarter, but because the condition that required arbitration no longer exists. The scheduler is not replaced by a better scheduler; it is replaced by the absence of the condition that made one necessary.

7. **Memory ordering is eliminated by default** — Memory barriers (`acquire`, `release`, `smp_mb()`, `READ_ONCE`, `std::atomic`) exist because the CPU and compiler reorder instructions and the programmer must constrain that reordering at runtime to prevent data races. Every barrier is a runtime substitute for a compile-time proof the system does not have. When every access cycle is declared and compiler-verified, ordering is a property of the schedule — if core 2 reads at cycle N and core 0 writes at cycle N−K where K > 0, the write provably precedes the read without a barrier, because the schedule makes it impossible for it not to. This eliminates memory ordering overhead at every layer: every `smp_mb()` in the kernel, every `std::atomic` acquire/release in userspace, every `rcu_dereference()` — all runtime substitutes for the same missing compile-time proof.

8. **A faster general-purpose kernel** — The RCU elimination claim is not limited to the isolated app partition. A mainstream Linux kernel annotated with clock-aware scheduling proofs would remove RCU overhead from its own hot paths: the scheduler runqueue, the routing table, the socket hash, the file descriptor table. These datasets are small and their access patterns are not inherently non-deterministic — they are protected by RCU because timing was never declared, not because timing is genuinely unknowable. Declaring it makes RCU unnecessary. The result is a leaner kernel for every workload, not just HFT.

---

## The Central Claim

**If Linux adopted clock-aware programming, it would be a fundamentally more efficient operating system.**

Not faster in one benchmark, not better in one workload — structurally more efficient across the board. Every RCU read-side critical section, every memory barrier, every scheduling decision, every interrupt deferral, every softirq, every lock is a runtime response to a single missing fact: when does this code run, and when does anything that could conflict with it run? Clock-aware annotations answer that question at compile time. The runtime machinery that exists to compensate for not knowing the answer becomes unnecessary — not deprecated, not replaced by a lighter version, but provably redundant.

The Linux kernel is not inefficient because its engineers made poor choices. It is the most sophisticated general-purpose OS ever built. It is inefficient in specific, structural ways that are a direct consequence of building without declared timing — and those inefficiencies compound: no timing → locks → RCU → forced indirection → heap allocation → TLB pressure → cache misses → more latency → more scheduling variance → more locks. Each layer adds overhead because the layer below it could not provide a timing guarantee.

Clock-aware programming is the missing primitive that breaks this chain at the root.

### Hardware-aware programming is the only genuinely efficient programming

Every abstraction that hides the hardware trades efficiency for convenience. Virtual memory hides physical addresses — and pays with TLB misses and page fault latency. The scheduler hides core assignment — and pays with preemption, context switch cost, and cache invalidation. RCU hides write timing from readers — and pays with grace period tracking, memory barriers, and forced pointer indirection. The OS network stack hides the NIC — and pays with interrupt delivery, sk_buff allocation, and multiple data copies.

None of these abstractions are wrong. They are the right tradeoff for general-purpose software where hardware details are genuinely unknown to the programmer. But they are not free, and their cost is not optional — it is structural. A system that hides the hardware pays for that hiding on every operation, forever.

Clock-aware programming is the observation that for a large and important class of code, the hardware details are *not* unknowable — the programmer knows exactly which core will run this code, at what frequency, with what working set. The details are known; they are just never declared. Declaring them allows the compiler to see the hardware directly, reason about it precisely, and eliminate every abstraction layer whose only purpose was to compensate for not knowing what the programmer already knows.

A kernel built on these principles — not as a research experiment but as a production engineering choice — would be the first operating system that is efficient by construction rather than efficient by tuning.

---

## CPU Partition Model

This architecture — isolated cores, poll-mode NIC, ring buffer boundary, `isolcpus`/`nohz_full` — is specifically for **HFT-class and hard real-time workloads** where the cost of a single scheduler preemption or IRQ delivery is unacceptable. Normal workloads do not need it and should not use it.

Clock-aware programming as a concept applies more broadly: the annotations, compile-time timing proofs, memory ordering elimination, and RCU elision are useful for any workload that benefits from statically verified timing. But those benefits do not require dedicated isolated cores. A web server or database annotating its hot path gets compile-time cycle budgets and provably unnecessary barriers without partitioning a single core away from the OS.

The partition model is the extreme end of the spectrum — the configuration for workloads where the OS must be structurally excluded, not just tuned.

This is a **software partition enforced by kernel configuration**, not a hardware boundary. On a normal system, all cores share timeslices and receive interrupts freely. The partition is created by directing IRQs away from the app cores (`irqaffinity`), suppressing the scheduler tick on them (`nohz_full`), removing them from the RCU quiescent-state tracking loop (`rcu_nocbs`), and marking them as isolated from the load balancer (`isolcpus`). Together these mechanisms approximate what a hard hardware partition would enforce — but it is a configuration discipline, not a silicon guarantee.

```
┌─────────────────────────────────────────────────────────────┐
│                        System                               │
│                                                             │
│  OS Partition (cores 0–1)        App Partition (cores 2–7)  │
│  ┌────────────────────────┐      ┌────────────────────────┐ │
│  │ Linux scheduler        │      │ Timeslice-managed FSM  │ │
│  │ IRQ handling           │      │ No OS intervention     │ │
│  │ RCU, memory reclaim    │      │ Poll-mode NIC access   │ │
│  │ Management plane       │      │ L1-pinned working set  │ │
│  └────────────┬───────────┘      └──────────┬─────────────┘ │
│               │   Shared ring buffer         │               │
│               │   (declared address,         │               │
│               └─────declared cycle)──────────┘               │
└─────────────────────────────────────────────────────────────┘

Enforced by: isolcpus, nohz_full, irqaffinity, rcu_nocbs
```

**Analogy:** ARM Processing System (OS) + Programmable Logic fabric (app) on a Xilinx Zynq — except the boundary is kernel configuration rather than silicon. On the FPGA, the PL fabric genuinely cannot be preempted by the PS; on the CPU, that property is approximated by routing all interrupts and scheduler decisions away from the app cores.

---

## Toolchain Vision

```
system.cap          →   kernel boot parameters (generated, not hand-written)
                    →   compiler verification scope

[[timeslice(4ns, cpu=skylake, working_set=L1)]]
                    →   IDE shows cycle count inline (red underline on budget violation)
                    →   llvm-mca integration, not a new analysis engine

Feedback loop today:   write → compile → run → profile → fix  (minutes per iteration)
Feedback loop target:  write → IDE annotation (instantaneous)
```

The goal is the same feedback loop a hardware engineer has in Vivado: timing violations are visible as you type, not discovered after a build.

---

## Why Rust

The Rust borrow checker already proves read/write exclusivity at compile time — the same invariant RCU enforces at runtime. Extending an existing proof system is feasible; building one from scratch is not. Lifetimes provide a natural hook for timeslice annotations: `'timeslice(N)` as a bound encoding "this reference is valid during cycle N on core K". The `rust-for-linux` project is an active community path to mainline kernel contribution.

See [ADR-0004](docs/adr/0004-rust-as-implementation-vehicle.md) for the full decision.

---

## Architecture Decision Records

These ADRs form the whitepaper. Each records a single decision: the problem it solves, the constraints it operates under, the rationale, and the alternatives rejected.

| # | Title | Status |
|---|---|---|
| [0001](docs/adr/0001-naming-clock-aware-programming.md) | Naming: clock-aware programming | Accepted |
| [0002](docs/adr/0002-cpu-partition-model.md) | CPU partition model (OS cores / app cores) | Accepted |
| [0003](docs/adr/0003-rcu-elimination-via-compile-time-proofs.md) | RCU elimination via compile-time scheduling proofs | Accepted |
| [0004](docs/adr/0004-rust-as-implementation-vehicle.md) | Rust as implementation vehicle | Accepted |
| [0005](docs/adr/0005-unified-system-configuration.md) | Unified system configuration file | Accepted |
| [0006](docs/adr/0006-poll-mode-self-regulating-network-stack.md) | Poll-mode self-regulating network stack | Accepted |
| [0007](docs/adr/0007-memory-ordering-elimination.md) | Memory ordering elimination via compile-time scheduling proofs | Accepted |
| [0008](docs/adr/0008-clock-aware-memory-management.md) | Clock-aware memory management | Accepted |
| [0009](docs/adr/0009-implied-hardware-architecture.md) | The implied hardware architecture | Speculative |

---

## The Implied Hardware Destination

Clock-aware programming is designed to run on existing commodity CPUs. But the model has a logical hardware endpoint that is worth stating explicitly, because it determines where the compounding value lives.

Every piece of hardware in a modern CPU that does not perform computation exists to compensate for not knowing the schedule: the out-of-order engine, the branch predictor, the ROB, the speculative execution machinery, the L2 and L3 cache hierarchy. A CPU designed for a software model that *declares* its schedule has no use for any of it. The compiler already scheduled optimally from the declared dependency graph. The branch-free hot path has no branches to predict. The L1-pinned working set has no misses for L2 and L3 to absorb.

The freed die area — L2 and L3 together consume 50–70% of a modern CPU die; the OOO engine and branch predictor consume much of the rest — is reinvested into execution ports, wider SIMD, more cores, and higher clock frequency. The memory model simplifies to TSO or stronger by construction, eliminating residual barriers and enabling full RCU elimination without architecture-dependent caveats.

The compounding gain comes from transistor scaling. As transistors shrink, every process node amplifies the freed area. An OOO CPU at 3nm is faster than at 5nm by a predictable factor. A clock-aware in-order CPU at 3nm is faster by a larger factor, because it reinvests *more* of the node's transistor budget into computation rather than prediction. The FPGA is the existence proof: FPGAs are in-order by construction, and every process generation has delivered gains directly into logic density and clock rate, not into prediction hardware.

This is not a requirement for the software model — which works on Skylake today. It is the destination the model implies, and the reason the primitive has value that compounds rather than saturates. See [ADR-0009](docs/adr/0009-implied-hardware-architecture.md).

---

## Applicability Beyond HFT

The ideas here were motivated by HFT and derived from FPGA work, but the missing primitive — declared access timing — is absent from every computing platform, and the problems it causes appear everywhere.

**Embedded systems** — Microcontrollers and RTOSes live with the same tradeoffs: FreeRTOS tasks have priorities and preemption because task duration is unknown; interrupt service routines are short and deferred because their timing conflicts with the main loop are unresolved at compile time; power management is heuristic because the system cannot predict when the next operation will run. Clock-aware annotations would make all of this static: task duration verified at compile time, ISR scheduling provably non-conflicting, sleep cycles declared rather than estimated.

**Mobile development** — Frame rendering, touch latency, audio callbacks, and background work all compete on a shared scheduler that cannot see across their boundaries. Jank, audio glitches, and battery drain under light load are symptoms of the same unknowing: the runtime arbitrates between operations whose timing was never declared. A clock-aware model would let the compiler prove that an audio callback completes within its 5 ms window, that a frame's render work finishes before the display's vsync deadline, and that background tasks back off precisely when foreground work begins — without the heuristics, tuning, and platform-specific workarounds that currently make mobile performance engineering an empirical discipline.

**Real-time systems and safety-critical software** — DO-178C, IEC 61508, and AUTOSAR all require worst-case execution time (WCET) analysis, typically done with external tools against pre-compiled binaries. Clock-aware annotations would make WCET a first-class compile-time output, not a post-hoc measurement — the certification artefact becomes a property of the source code, not of one specific compiled binary on one specific hardware configuration.

**Garbage collection** — A clock-aware GC would be the most efficient GC ever built. Today's GC designs — generational, concurrent, incremental, ZGC, Shenandoah — all exist to manage one fundamental problem: the collector does not know when the mutator will next access a given object, so it must either stop the world (pause) or use complex concurrent barriers to stay correct while the mutator runs. Every GC barrier, every write barrier, every remembered set, every card table entry is a runtime mechanism for tracking what the compiler could have declared statically.

A clock-aware GC declares when the mutator accesses which memory regions and at what cycle windows collection can run safely. Collection is not scheduled heuristically based on heap fill — it is declared at a cycle boundary where the mutator is provably not accessing collectable memory. No stop-the-world pause because the collection window is a scheduled timeslice, not an interruption. No concurrent barriers because the compiler has already proved the mutator and collector access non-overlapping cycles. No write barriers because pointer updates happen at declared cycles that the collector knows to avoid.

The heap itself becomes a declared memory partition: allocation regions, survivor regions, and collection windows are all compile-time declarations rather than runtime heuristics. The GC stops being a background thread hoping to keep up and becomes a first-class participant in the program's declared schedule.

At the cycle boundary, the circuit asks two questions — and only two:

1. Did the thread request more memory this cycle?
2. Did the thread finish its execution this cycle?

If yes to (2): reclaim. Immediately, at the boundary, with no grace period, no barrier, no concurrent marking phase — because the compiler already proved at build time that no other thread holds a reference to that memory past this cycle. Memory management collapses to a per-cycle state machine with two states. Every GC algorithm ever invented is a runtime approximation of this two-question circuit.

In each domain, the current answer is a runtime mechanism (scheduler, priority system, WCET tool) that solves at runtime what declared timing would prevent at compile time.

**Power consumption** is the natural concern with a model built on dedicated cores and polling loops: a core that spins looks like a core that burns. The self-regulating backoff (ADR-0006) is the direct answer. Under low load the poll interval expands by a declared number of cycles — the core is genuinely idle between checks, not spinning. The idle period is not a heuristic sleep or a timer interrupt guess; it is a compiler-known cycle count derived from the ring depth at the last boundary. The core wakes exactly when it declared it would, does exactly the work that was budgeted, and returns to idle. This is closer to a clock-gated circuit than to a busy-wait loop: activity is proportional to actual work, transitions are instantaneous at the cycle boundary, and the power envelope is as predictable as the timing envelope — because they are the same thing.

The mobile modem is a concrete example of the leverage. A modem is a real-time pipeline — channel decoding, demodulation, protocol state machines — running on dedicated DSP or application processor cores alongside the general-purpose OS. Today, power management for the modem is a stack of heuristics: radio frequency duty cycling, DRX (discontinuous reception) timers tuned per operator, proprietary firmware that the OS cannot inspect. The result is a permanently negotiated truce between latency and battery life, re-tuned for every chipset and every carrier. Under a clock-aware model, the modem pipeline declares its cycle budget per frame interval (e.g. every 1 ms LTE subframe). Between frame boundaries the cores are idle for a compiler-proven duration — not an estimated one. The radio wakes at the declared cycle, processes the frame within the declared budget, and returns to a known-idle state. Power draw becomes a compile-time output of the frame schedule, not a runtime measurement that firmware attempts to minimise after the fact. The same model that removes the Linux scheduler from a trading system removes the DRX heuristic from a modem.

---

## The Simplification Thesis

Every complex algorithm in systems programming exists to compensate for not knowing when things happen.

- **RCU** — complex because the writer does not know when readers finish.
- **Generational GC** — complex because the collector does not know when the mutator will next touch an object.
- **NAPI / softirq / sk_buff** — complex because the kernel does not know when the next packet arrives or when the application will read it.
- **CFS scheduler** — complex because the OS does not know how long each task will run.
- **Memory barriers / `seq_cst`** — complex because the compiler does not know the relative ordering of accesses across cores.
- **DRX power management** — complex because the firmware does not know when the next frame will arrive.
- **WCET analysis tools** — complex because the certification body does not know worst-case execution time from the source code.
- **Lock-free queues, hazard pointers, epoch reclamation** — complex because producers and consumers do not know each other's timing.

In every case: remove the unknowing, and the algorithm that compensates for it becomes unnecessary. The complexity is not intrinsic to the problem — it is intrinsic to the absence of declared timing. Clock-aware programming does not replace these algorithms with better algorithms. It removes the condition that made them necessary. The simplification is not incremental; it is structural. Declare when things happen, and the entire edifice of compensating machinery dissolves.

### What about cache management?

The objection is natural: doesn't a polling loop or a deterministic access pattern still require explicit cache management — prefetch instructions, cache-line alignment, working set pinning?

The answer is that a fully deterministic access schedule solves cache management as a consequence, not as an additional concern. The hardware prefetcher works by detecting stride patterns in memory accesses. A clock-aware access schedule *is* a stride pattern — the same memory regions are accessed in the same order at the same cycle interval, every iteration, without exception. The prefetcher has the simplest possible job: a perfectly regular pattern it can detect after one or two iterations and predict indefinitely.

On an FPGA this is not a concern at all — BRAM is accessed at declared addresses at declared clock cycles; there is no cache, no prefetcher, no miss penalty. The clock-aware CPU model approximates this: the working set is declared and pinned in L1 (`mlock`, huge pages), the access pattern is regular by construction, and the hardware prefetcher operates on a schedule it can fully anticipate. Explicit `__builtin_prefetch` calls are an optimisation for *irregular* access patterns — they are unnecessary when the pattern is declared and regular. The determinism that eliminates RCU and memory barriers is the same determinism that makes the prefetcher optimal without intervention.

### A new class of higher-level languages becomes possible

The simplification thesis has an upward consequence that is as significant as the downward one.

Today, every high-level language pays a systems tax. Python, Java, Go, Swift — all of them run on runtimes that have the same unknown-timing problem at their foundation: a GC that does not know when the mutator will access memory, a scheduler that does not know when tasks will complete, memory barriers in the runtime's own data structures. The high-level language is expressive and the programmer is productive, but the efficiency floor is set by the runtime's compensating machinery, not by the programmer's intent. No amount of language-level optimisation escapes the cost that the runtime pays for not knowing when things happen.

A clock-aware foundation changes that floor. If the runtime declares its own timing — when the GC runs, when the scheduler tick fires, when cross-thread data becomes visible — the compensating machinery in the runtime dissolves by the same logic as it dissolves in the kernel. A high-level language built on a clock-aware runtime inherits structural efficiency without requiring the programmer to reason about cycles at all. The cycle declarations live in the runtime and the standard library; the application programmer writes expressive, high-level code; the compiler verifies the timing contract end-to-end.

This is a class of languages that does not currently exist: **high-level and structurally efficient by construction**, not by tuning. Today the choice is between expressiveness (managed languages with GC and a scheduler) and efficiency (systems languages with manual memory management and explicit synchronisation). Clock-aware programming suggests that choice is not fundamental — it is a consequence of the missing timing primitive. Declare the timing at the foundation and the tradeoff dissolves. A language can be as expressive as Python and as structurally efficient as the hardware allows, because the runtime it runs on finally tells the truth about time.

---

## The Hardest Objection: How Do We Guarantee the Cycles?

The most serious challenge to this model is also the most obvious one: **`llvm-mca` predicts cycle counts from a microarchitecture model. At runtime, the actual CPU may not match that prediction.** Branch mispredictions, unexpected cache misses, microcode updates, frequency transitions, TLB shootdowns — any of these can cause actual execution to exceed the declared budget. If that happens, the compile-time proof is invalid. Is this an idea problem, or a compiler problem?

It is a **compiler accuracy problem, not an idea problem** — and it is the same problem FPGA timing closure solves.

### The FPGA parallel

In Vivado, a timing constraint (`set_max_delay 4ns`) is verified against a model of the silicon: gate delays, interconnect capacitance, temperature corner. The model is not perfect — silicon variation, temperature drift, and aging all cause actual propagation times to deviate from the model. The solution is:

1. **Margin** — the constraint target includes slack. A 4 ns budget is not "exactly 4 ns"; it is "4 ns with enough margin that the worst-case silicon and temperature variation still meets it."
2. **Corner analysis** — the toolchain verifies timing at slow/fast/nominal corners, not just the nominal model.
3. **Runtime monitoring** — some designs include built-in self-test circuits that measure actual timing margins on the deployed silicon.

Clock-aware programming uses the same three mechanisms.

### The theoretical ceiling is known

Every CPU microarchitecture publishes its throughput model: instructions per cycle per execution port, latency per instruction class, store-to-load forwarding latency, cache access times. Intel's Optimization Reference Manual, AMD's Software Optimization Guide, ARM's Cortex technical reference manuals — all of these encode the theoretical maximum performance for any instruction sequence on a given microarchitecture. `llvm-mca` is a direct consumer of these specifications.

This means the ceiling is not an empirical question. A declared `cpu_model` in `system.cap` fixes the ceiling exactly. The compiler checks the instruction mix against the published model and verifies or rejects the annotation. The remaining variance between model and silicon — the gap the manufacturer's specification does not cover — is narrow, bounded, and well-characterised for modern server-class CPUs at pinned frequency on L1-resident data. It is the margin.

### Margin: the budget is a bound, not a point

`[[timeslice(budget_ns = 4)]]` declares a budget of 4 ns. The compiler verifies the worst-case instruction path — the maximum cycle count under the `llvm-mca` model — fits within 4 ns. The developer chooses the budget with margin: if `llvm-mca` reports 3.1 ns worst-case, declaring 4 ns provides ~0.9 ns of margin for microarchitectural variance. The margin is explicit and visible in the annotation. This is directly analogous to timing slack in digital design.

### Preconditions eliminate the main sources of variance

The preconditions in ADR-0002 are not arbitrary — they are specifically chosen because they are the conditions under which `llvm-mca` is most accurate:

| Source of variance | Precondition that eliminates it |
|---|---|
| Cache miss latency variance | Working set declared and pinned in L1 (`mlock`, huge pages) |
| Branch misprediction | Branch-free hot path (compiler intrinsics, cmov) |
| Frequency scaling / turbo | CPU frequency pinned (`cpupower`, `cpu_freq_mhz` in `system.cap`) |
| Scheduler preemption | `isolcpus`, `nohz_full` |
| IRQ delivery | `irqaffinity` |
| TLB shootdowns | Huge pages, no concurrent mapping changes on app-partition memory |
| Hyperthreading interference | Whole physical core isolated (not just logical core) |

Under all of these preconditions, `llvm-mca`'s prediction accuracy is as high as it can be. The remaining variance is microcode-level behavior that is bounded and well-characterised for each `cpu_model`. This is exactly the operating regime HFT shops already rely on for sub-microsecond latency guarantees — the model is not theoretical.

### Runtime monitoring as the feedback loop

The runtime configuration library (ADR-0005) closes the loop. The OS partition can observe actual execution times (via hardware performance counters: `PERF_COUNT_HW_INSTRUCTIONS`, `PERF_COUNT_HW_CPU_CYCLES`) and compare them against declared budgets. If actual cycles consistently exceed the declared budget, the system writes an updated budget via the config ring at the next cycle boundary — tightening the declared budget to match reality, or alerting the operator that the working set no longer fits in L1.

This is not a violation of the compile-time proof; it is the monitoring layer confirming the proof's preconditions hold. If they do not hold (e.g., a cache miss appears because the working set grew beyond L1), the system self-reports the violation rather than silently producing incorrect timing behavior.

### What this model does not claim

It does not claim that declared cycle counts are physically guaranteed the way an FPGA clock period is guaranteed after timing closure. CPUs are more complex than FPGAs: out-of-order execution, speculative loads, and microcode have interactions that no static model captures perfectly.

What it claims is: **under the declared preconditions, with declared margin, the compile-time prediction is accurate enough to replace runtime synchronisation mechanisms whose correctness requirements are far less strict.** RCU does not need cycle-perfect timing — it needs proof that reads and writes do not overlap. A 0.9 ns margin on a 4 ns budget means the proof holds even if the actual execution is 9 ns longer than the nominal model predicts, which is conservative for any L1-resident, branch-free hot path on a pinned-frequency core.

The compiler problem is real. It is the implementation challenge. The idea is sound.

---

## Origin

This work is derived from hands-on experience building two systems:

- **fpga-feed-handler** — NASDAQ ITCH 5.0 feed handler in Vitis HLS. The FPGA work is the existence proof that the model produces correct, verifiable, deterministic systems. Clock constraints are not a theoretical aspiration; they are a daily engineering discipline.
- **low-latency-feed-handler** — DPDK-based feed handler in C++23. The exercise of eliminating the Linux network stack in software made the parallel to FPGA execution visceral.

Both systems were built self-taught. The ideas here did not come from a compiler research group or an OS lab — they came from the experience of building real hardware and real software at the latency boundary and noticing that the hardware world had already solved the problem the software world was still approximating with heuristics.

HLS for FPGAs is a particular kind of discipline. Even in a high-level synthesis flow, you reason about cycles explicitly: how many clock cycles does this loop iteration take, does this signal arrive in time for the next stage to consume it, where does the pipeline stall, what is the initiation interval. The toolchain enforces these — a timing violation is not a warning, it is a synthesis failure. That constant contact with cycle-level thinking, combined with building a DPDK-based system where the same questions recur in software without any toolchain enforcement, is what produced the core observation: the discipline exists in hardware; the primitive is missing in software.

Background reading:

- Robert Love, *Linux Kernel Development* — RCU, locking, preemption chapters. The reference for the kernel internals analysis in ADR-0003.
- Paul McKenney, *Is Parallel Programming Hard, And, If So, What Can You Do About It?* — the authoritative treatment of RCU by its principal designer. The framing of RCU as deferred reclamation under reader/writer timing uncertainty is his.
- Ulrich Drepper, *What Every Programmer Should Know About Memory* — cache hierarchy, TLB behaviour, NUMA topology, prefetcher mechanics. The reference for the memory partition and working set claims.
- Agner Fog, *Microarchitecture* — the most detailed published treatment of CPU execution ports, instruction latency, and throughput per microarchitecture. The empirical basis for the claim that the theoretical ceiling is known and documented.
- David Harris & Sarah Harris, *Digital Design and Computer Architecture* — the foundational reference for pipeline design, clock domain crossings, and synchronous circuit timing. The conceptual bridge between FPGA design discipline and the clock-aware programming model.
- Scott Meyers, *Effective Modern C++* — the practical reference for modern C++ memory model semantics, `std::atomic`, move semantics, and the reasoning patterns that made the memory ordering analysis in ADR-0007 precise.

---

## Acknowledgements

To Robert Love, Paul McKenney, Ulrich Drepper, Agner Fog, David and Sarah Harris, and Scott Meyers: your writing made the ideas in this repository possible. None of this would exist without the depth and rigour you put into work you made available.
To Linus Torvalds and the Linux kernel maintainers: the kernel you built and keep building is the system this work proposes to improve. The fact that it is open, auditable, and understood well enough to reason about at the level of individual hot paths is itself a precondition for everything in these ADRs. The critical analysis here is the highest form of respect — you built something worth analysing.

To Bjarne Stroustrup and the C++ standards committee: `low-latency-feed-handler` is C++23. The language gave the tools — concepts, ranges, `std::atomic`, the memory model — to build a system where the questions about timing became sharp enough to ask. The sharpness of the question is partly the language's doing.

To the Xilinx (now AMD) Vivado, Vitis HLS, and High-Level Synthesis teams: the HLS toolchain is where the cycle-level thinking became non-negotiable. A timing violation in Vivado is not a warning — it is a synthesis failure. That discipline, enforced by your tools on every compile, is what made the absence of the same discipline in software impossible to ignore. `fpga-feed-handler` exists because your toolchain made it buildable. The core observation in this repository exists because your toolchain made cycle-awareness unavoidable.
To the Microsoft and GitHub Copilot teams: this repository was built under real constraints — a busy life, a two-year-old, and very little uninterrupted time. Copilot was the tool that made it possible to ship a complete, coherent body of work in the gaps. The ideas are mine; the bandwidth to articulate them properly came from having the right tool at the right moment. Thank you.

