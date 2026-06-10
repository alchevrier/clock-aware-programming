# Paper IV: Hardware Architecture Implications
#### Alex Chevrier — chevrier.alex@gmail.com

> A CPU designed for software that declares its schedule has no use for an out-of-order engine, branch predictor, or L2/L3 cache hierarchy. When the schedule is known, every transistor dedicated to prediction is a transistor wasted.

This paper follows from [Paper I](01-clock-aware-programming.md), [Paper II](02-language.md), and [Paper III](03-os-and-runtime.md). The clock-aware primitive has consequences that propagate downward into silicon: a software model that declares its timing does not need hardware that compensates for undeclared timing. This paper describes the hardware that clock-aware programming implies.

---

## The Starting Point

Paper I establishes that under a set of achievable preconditions — L1-pinned working set, isolated core, branch-free hot path, channel-based I/O — a CPU core can be made to behave like FPGA fabric: its cycle count is a pure function of instruction mix × microarchitecture model, computable at compile time.

Paper II establishes that the language has four rules — clock annotation, lifetime type, channel, exhaustive match — and that every circuit is hardware-correct by construction.

Paper III establishes that when the OS is written in the same language, the runtime, kernel, scheduler, memory manager, and driver model all collapse into the same model: declared-timing circuits, slotted into a static dispatch table.

Paper IV asks: what hardware would you design for this model? Not "what hardware does it run on today" — it runs on Skylake today, by design — but what hardware does the model *imply*, if the silicon designer starts from the software model rather than from legacy compatibility requirements?

---

## What the Hardware Is Compensating For

Every component in a modern CPU that does not perform computation exists to compensate for not knowing the schedule.

| Hardware component | What it compensates for |
|---|---|
| Out-of-order execution engine | Instructions arrive in program order but execute optimally in a different order — because the compiler didn't know the hardware's port topology when it emitted code |
| Branch predictor | The next instruction is not known until the branch resolves — because branches encode undeclared control-flow decisions |
| Reorder buffer (ROB) | Instructions must retire in order despite executing out-of-order — a buffer to hold the inflight window |
| Speculative execution | Work is performed before it's known to be needed — to hide the latency of not knowing |
| L2 and L3 cache | Data that doesn't fit in L1 must be fetched — because access patterns are not declared, so L1 cannot be sized for the working set |
| Cache coherency hardware | Multiple cores may access the same cache lines — because the access schedule is not declared, so the hardware cannot know they don't |
| Memory ordering hardware | Stores and loads may be reordered by the hardware — the hardware must track this so software can insert barriers to constrain it |
| TLB and page table walker | Virtual-to-physical translation on every access — because the OS hides physical addresses and the application cannot declare them |

Every item on this list is a hardware mechanism whose purpose is to produce correct results when the software did not tell the hardware what was going to happen. The mechanism is not computation. It is compensation.

A clock-aware CPU — one designed for software that declares its schedule — has no use for any of them.

---

## One Architecture

The clock-aware hardware architecture is defined by what it removes.

### Unified Memory — A Motherboard-Level Standard

No VRAM. No PCIe copies. No DRAM/cache split that the programmer must reason about.

A conventional system has at least three distinct memory pools: system DRAM, GPU VRAM, and the CPU cache hierarchy. Moving data between them requires explicit copies, DMA transfers, or cache coherency protocols — all of which consume bandwidth, introduce latency, and require programmer or runtime intervention.

Unified memory already exists in limited form (Apple M-series), but it still requires coherency hardware — because timing is not declared. When two cores can access the same cache line at any undeclared time, the hardware must track that possibility and resolve it dynamically. Coherency hardware is the runtime mechanism that compensates for not knowing when each device will access each region.

Clock-aware programming removes the last reason for separated memory pools. When every device's access window is declared, the hardware knows at build time that no two devices will access the same region at the same time. There is nothing for coherency hardware to resolve. The pools can be physically unified without a coherency protocol — not as a design compromise, but as a provable consequence of the declarations.

This is clock-aware programming applied to the motherboard layer. The same principle — declare the timing, eliminate the compensating mechanism — applied one level down, to the physical interconnect between CPU, GPU, NIC, and NVMe. The implications:

- **Zero copy between CPU, GPU, NIC, NVMe** — every device reads from and writes to the same physical pool at declared cycles
- **PCIe becomes unnecessary for data movement** — PCIe exists to move data between separate pools across a high-latency interconnect; when there is one pool, the interconnect carries only control signals
- **No coherency silicon** — the freed die area in every device on the board is reinvested into computation
- **Power savings across the board** — PCIe link power, coherency snoop traffic, and inter-pool copy operations are all eliminated

A motherboard standard that defines the physical and timing interface for this unified pool — the clock distribution, the declared access window protocol, the memory map — is the hardware complement to the software declaration model. The standard is not defined by instruction sets or bus protocols. It is defined by what the software declares.

In a clock-aware system, memory is one pool with declared tiers:

```
Tier 0: L1 — working set for the current circuit, declared and pinned (< 32 KB)
Tier 1: L2 — inter-circuit buffers, declared access windows
Tier 2: Declared heap — lifetime-typed allocations, known at compile time
```

There is no "GPU memory" because there is no GPU — the processing elements are the in-order execution cores. There is no "cache miss" because the working set is declared and pinned. The memory hierarchy exists, but it is not a source of variance — it is a set of declared tiers with known latencies, verified at compile time.

### Declared Access Windows — No Coherency Hardware

Cache coherency hardware (MESI protocol, coherency directories, inter-core snoop traffic) exists to maintain consistency between cores accessing the same data at unknown times. If core 0 and core 2 might both read or write the same cache line at any time, the hardware must track this and enforce consistency dynamically.

In a clock-aware system, every core's access window is declared. Core 0 writes at cycle N. Core 2 reads at cycle N+K where K > write latency. The compiler has proved this ordering. There is nothing for coherency hardware to resolve — the ordering is established at compile time. The cache lines are owned by one circuit at a time, by declaration.

No coherency hardware. No snoop traffic. No MESI state machine. The freed transistors are execution units.

### Layered Memory Tiers — Sized by Declared Lifetimes

Today, L2 and L3 caches are sized as compromise guesses: large enough to hold the working set of typical programs that did not declare their working set. L3 on a modern server CPU is 32–96 MB — an enormous structure whose purpose is to compensate for the fact that no program told the hardware what it needed to keep resident.

In a clock-aware system, every circuit declares its working set. L1 is sized for the declared working set of the most demanding circuit. There is no working set beyond that declaration because the model does not permit undeclared access. L2 is sized for inter-circuit buffers — channel depths, declared explicitly in `system.cap`. L3 disappears: it was a buffer for accesses that should not have happened.

The die area freed by eliminating L3 (30–40% of a modern CPU die) is reinvested into execution ports and clock distribution.

### Clock Distribution — The Global Synchroniser

On an FPGA, the clock is the global synchronisation primitive. Every register is clocked. Every signal crosses clock domain boundaries at declared points, verified by the timing closure tool. The clock is not a performance feature — it is the correctness foundation.

On a clock-aware CPU, the same principle applies. The clock is the schedule. Every circuit executes at its declared cycle. Every channel write happens at a declared clock edge. Every channel read happens at a declared clock edge. The clock distribution network is designed for uniform arrival time across all execution cores — not as a performance optimisation, but as a correctness requirement. The declared ordering is only guaranteed if the clock arrives at all cores within the declared timing margin.

This is the same engineering discipline as FPGA clock distribution. The toolchain verifies clock arrival skew as part of timing closure.

### Not x86. Not ARM. Not RISC-V.

The clock-aware architecture is not an extension of an existing ISA. It is defined by its declaration model, not by its instruction set. Any silicon can implement it — because the definition is in what the software declares, not in what instructions the hardware executes.

This is intentional. An ISA-first architecture is constrained by legacy compatibility: every new feature must coexist with the instruction encodings, memory models, and privilege levels of the existing ISA. A declaration-first architecture has no such constraint. The instruction set is the minimum required to implement the declared operations efficiently. It is defined by the model, not inherited from history.

In practice, an initial implementation targets an existing ISA (x86-64 or AArch64 — both are viable) because it runs on commodity hardware without custom silicon. The long-term architecture is defined by the model. Any instruction set that efficiently implements channel reads, cycle-timed execution, and L1-tier memory access is a valid implementation. The ISA is an implementation detail. The declaration model is the architecture.

---

## The Die Area Reallocation

On a modern server CPU (Skylake, Zen 4, Neoverse), the die area breakdown is approximately:

| Component | Approximate die area |
|---|---|
| L2 and L3 cache | 50–70% |
| Out-of-order execution engine (ROB, RS, rename) | 10–15% |
| Branch predictor | 5–10% |
| Front-end (fetch, decode, BTB) | 5–8% |
| Execution units (ALU, FPU, SIMD) | 10–15% |
| Memory subsystem (TLB, LSU) | 5–8% |

A clock-aware CPU eliminates the first four categories entirely: L2/L3 (replaced by declared tiers), OOO engine (not needed when the compiler scheduled optimally), branch predictor (not needed when branches are eliminated on the hot path by declaration), and the front-end complexity that supports speculation.

The freed area — 70–90% of the current die — is reinvested into:

- **More execution ports** — wider SIMD, more parallel ALU, higher throughput per core
- **More cores** — the same die area yields 5–10× more cores at similar clock rate
- **Higher clock frequency** — less thermal density from eliminated prediction hardware; the freed thermal budget goes into frequency
- **Larger declared L1** — the working set declaration allows the L1 to be sized precisely for the application class, rather than as a general-purpose guess

The result is a processor that performs more computation per transistor than any OOO processor — because every transistor performs computation, and none performs prediction.

---

## The Compounding Gain from Process Scaling

The most significant property of this die area reallocation is that it compounds with process node scaling.

A conventional OOO CPU gains from a new process node in proportion to the transistors devoted to execution: roughly 10–15% of the die. The rest of the gain — from L3, from prediction hardware, from OOO engines — is spent on maintaining the same level of compensation at smaller feature sizes.

A clock-aware in-order CPU gains from a new process node in proportion to the transistors devoted to execution: 90–100% of the die. Every process node delivers its full transistor budget to computation.

The FPGA is the existence proof. FPGAs are in-order by construction. Every FPGA process generation has delivered gains primarily into logic density and clock rate — because there is no prediction hardware to re-implement. A 3nm FPGA has dramatically more logic than a 5nm FPGA; a 3nm OOO CPU has only marginally more useful compute than a 5nm OOO CPU, because the process budget was spent re-implementing the prediction machinery at the new node.

Clock-aware CPUs follow the FPGA scaling curve, not the OOO CPU scaling curve. At each process node, the gap between them widens.

---

## Hardware Cost Implications

The hardware cost of an AI inference system today is dominated by GPU VRAM. A system capable of running a 70B parameter model requires either:

- A discrete GPU with 80GB VRAM (NVIDIA A100): ~HK$75,000 per card
- Two discrete GPUs with 40GB VRAM each: ~HK$60,000 per pair
- An Apple M-series chip with unified memory: ~HK$40,000 (consumer laptop)

The Apple M-series is cheaper because unified memory removes the VRAM/DRAM boundary. But the M-series is still an OOO CPU — it retains the prediction hardware. It is cheaper because the architectural constraint (unified memory) was chosen, not because the OOO overhead was eliminated.

A clock-aware CPU with declared memory tiers eliminates the boundary architecturally. The inference engine and the application processor share the same memory pool — not as an engineering compromise, but as a consequence of the model: all access patterns are declared, so there is no reason to separate the pools. The hardware cost of running a large model is the hardware cost of enough DRAM to hold the weights, plus the execution units to process them. No discrete GPU. No PCIe interconnect. No memory copies across the boundary.

The cost reduction from OOO elimination and VRAM/DRAM unification, combined with the higher core count per die area, implies a 3–4× reduction in hardware cost for equivalent inference throughput compared to a GPU-based system. Not because the hardware became cheaper per transistor — it did not — but because the transistors that remain all perform computation rather than prediction.

---

## The FPGA as Existence Proof

FPGAs are the existence proof that this architecture is not theoretical.

An FPGA is in-order by construction. Every register is clocked. Every signal path has a declared timing constraint. Every access to block RAM is at a declared address at a declared clock edge. There is no out-of-order engine, no branch predictor, no L3 cache, no coherency protocol. And FPGAs are used in production today for exactly the workloads where declared timing matters most: HFT feed handlers, radio baseband processing, radar signal processing, network packet forwarding.

The objection is: FPGAs are slow per-gate compared to ASICs and even CPUs. This is true. But the slowness is not inherent to the in-order model — it is inherent to the reconfigurability overhead of FPGAs (LUT routing, programmable interconnect). An in-order ASIC is not slow. A modern CPU's execution units, stripped of the prediction machinery and running at the same frequency, are fast. The FPGA demonstrates that the model produces correct, verifiable, deterministic systems; custom silicon demonstrates that the model is fast.

The clock-aware CPU is the FPGA model in fixed silicon: the declaration discipline of an FPGA, the execution speed of a custom ASIC, the programmability of a general-purpose processor.

---

## The Software Model Determines the Hardware

The deeper observation is directional: the software model determines what hardware is necessary.

Today, hardware is designed first — x86, ARMv9, RISC-V — and software is adapted to run on it. The hardware makes assumptions (programs have branches, programs access memory unpredictably, programs run concurrently on the same cache lines) and builds mechanisms to handle those assumptions efficiently. The software inherits the assumptions.

Clock-aware programming inverts this. The software model declares what it needs: declared timing, declared access windows, declared channel topology, declared working set. The hardware that satisfies those declarations is provably simpler than a general-purpose OOO processor — because the declarations replace the hardware's compensating mechanisms with compile-time proofs.

This means the hardware specification is derivable from the software model. It is not a new ISA designed by intuition and committee. It is the silicon implementation of the verification domain: exactly the hardware needed to execute the declared schedule at the declared timing, no more.

A new standard that any silicon implementation can meet — because the standard is defined by the declarations, not by the instruction encodings. The ISA is a consequence of the model. The model is the architecture.

---

## New RAM Technologies This Unlocks

Today every system uses one type of DRAM for everything. It is a compromise for all workloads — sized for the worst case, priced for volume, optimised for no particular access pattern.

Declared lifetimes change this. When the compiler knows at build time exactly how long every value lives and exactly which memory tier it occupies, the hardware can be heterogeneous — each tier implemented with the technology optimally suited to it, sized exactly for the declared working set, not as a general-purpose guess.

| Tier | Technology | Latency | Cost | Sized by |
|---|---|---|---|---|
| `Register` / `Ephemeral` | SRAM (registers, L1) | ~4 cycles | Very high per bit | Declared ephemeral working set — stays small |
| `Task` / `Warm` | HBM4 (High Bandwidth Memory) | ~10 cycles | High, high bandwidth | Declared hot working set — exactly as large as needed, no waste |
| `Session` / `Resident` | LPDDR6 (Low Power DDR) | ~100 cycles | Low, low power | Everything that lives long |
| `Cold` | NAND / NVMe | ~10,000 cycles | Very cheap per GB | Loaded at declared cycle — no surprise page faults, no demand-paging latency |

Today HBM is used in high-end GPUs and AI accelerators — at enormous cost, because the entire VRAM pool must be HBM to avoid the worst-case miss. With declared lifetimes, only the declared `Task`-scoped working set needs HBM — a fraction of the total memory footprint. The rest is LPDDR6, which is cheap and power-efficient. The system pays for high-bandwidth memory only where the declarations prove it is needed.

The `Cold` tier is the most significant new capability. No managed language can express "load this value from storage at a declared cycle with a declared latency budget." Page faults are invisible, unpredictable, and cannot be proven absent. In the clock-aware model, a `Cold` access is declared, its load cycle is verified against the latency of the NVMe tier in `system.cap`, and the compiler rejects declarations whose timing budget is insufficient to cover the load. Storage latency becomes a compile-time constraint, not a runtime surprise.

The result is a system where every byte of memory is in exactly the right tier, consuming exactly the right amount of power and cost for its declared lifetime. Not as a tuning exercise. By construction.

---

## Relation to ADR-0009

ADR-0009 (The Implied Hardware Architecture) in this repository records the same observation in decision form: that the clock-aware software model implies a hardware architecture, and that the compounding value of the primitive lives in the hardware convergence rather than in the software optimisation alone. This paper expands that observation into a full analysis.

The current implementation targets commodity x86-64 and AArch64 hardware — the near-term path is practical and does not require custom silicon. ADR-0009 and this paper describe the endpoint the model implies, not the starting point the implementation requires.

---

*Part of the clock-aware programming series. See [Paper I: Clock-Aware Programming](01-clock-aware-programming.md) for the core primitive. See [Paper II: The Language](02-language.md) and [Paper III: The OS and Runtime](03-os-and-runtime.md) for the software stack consequences.*
