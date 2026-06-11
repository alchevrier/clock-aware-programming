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

**The root cause of the over-provisioning is the OS stall.** When the OS stalls — context switches, syscall latency, GC pause, scheduler jitter, interrupt handling, page fault — the CPU cannot do useful work. The hardware has no way to know the stall is coming, so it must be provisioned for the peak throughput the application demands, not the average throughput it actually delivers. The L3 cache is large because the OS might evict working sets during a stall. The OOO engine is deep because the OS might delay instruction delivery. The branch predictor is complex because control flow through the OS is unpredictable. Every layer of hardware compensation exists, in part, to recover performance that the OS stole.

In the clock-aware model there are no stalls. There is no scheduler jitter because the scheduler is the clock. There is no GC pause because there is no GC. There is no syscall latency because there are no syscalls. There is no page fault because every access is declared. The hardware does not need to compensate for OS behaviour it cannot predict — because the OS behaviour is fully declared and already proven not to interfere with the circuit's declared windows. The hardware schematics simplify dramatically:

| Hardware component | Conventional purpose | Clock-aware fate |
|---|---|---|
| L3 cache (50–60% of die) | Buffer against OS-induced working set eviction | Eliminated — working set is L1-pinned by declaration |
| OOO engine (10–15%) | Recover IPC lost to pipeline stalls from unpredictable control flow | Eliminated — instruction order is compiler-optimal |
| Branch predictor (5–10%) | Compensate for branches the compiler never resolved | Eliminated — hot path is branch-free by construction |
| Deep ROB / reservation station | Hide latency from memory stalls and mispredictions | Eliminated — no stalls, no mispredictions |
| TLB (virtual memory hardware) | Support address space isolation between OS processes | Eliminated — circuits use declared physical tiers, no virtual addresses |
| Interrupt controller complexity | Deliver async hardware events to an OS that isn't listening | Simplified to DREQ → DMA → channel write — no async delivery |

The CPU that remains is not a simplified version of a modern CPU. It is a different class of machine: a deep execution pipeline with a large register file, wide issue ports, and a precisely-sized L1 — nothing else. The schematic is simpler than anything built since the early RISC era, but faster than anything that has ever run, because every transistor it has is doing computation.

**The same logic applies to core count.** Today's CPUs need fewer, higher-quality cores — and those cores need to be engineered to extreme performance — precisely because each core gets stalled. When a core stalls, it stops contributing. To hit a throughput target, you need enough cores that the surviving non-stalled cores can cover for the ones that are waiting. The response to stall-induced throughput loss is to make each individual core faster and more resilient — deeper OOO window, larger ROB, higher clock rate, more speculative headroom — so it recovers faster from each stall. The core becomes more complex and more expensive to manufacture because it must compensate for the OS's behaviour.

In the clock-aware model, cores do not stall. A core that does not stall does not need a deep OOO window to recover. It does not need a large ROB. It does not need aggressive branch prediction. It needs a deep pipeline and wide execution ports — which are simpler to manufacture, cheaper per core, and have better yield because the transistor structures are regular and predictable rather than the irregular, state-heavy logic of a speculative engine.

The manufacturing consequence is direct: the same die budget that produces 16 high-quality, stall-tolerant OOO cores today produces 80–120 simpler, stall-free clock-aware cores. Each individual core is less powerful in isolation. But it never stalls — so it delivers its declared throughput continuously, not intermittently. 80 cores running continuously at their declared rate outperform 16 cores running at peak rate between stalls by a margin that the atom stream can prove, tick by tick.

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

### Speculative Cost After Adopting the Clock-Aware Model

The following is a speculative but derivable estimate of what equivalent inference capability costs once the clock-aware architecture is in production silicon. The baseline is a 70B parameter model at production serving throughput.

| System | Configuration | Estimated cost | Bottleneck |
|---|---|---|---|
| **Today — GPU-based** | 2× NVIDIA A100 80GB + host server | ~HK$180,000 | VRAM capacity, PCIe bandwidth, OOO overhead |
| **Today — Apple M-series** | Mac Studio Ultra (192GB unified) | ~HK$40,000 | OOO overhead, memory bandwidth shared with CPU |
| **Clock-aware Gen 1** *(commodity silicon, software model only)* | Existing AArch64 server, clock-aware runtime, no custom silicon | ~HK$15,000 | ISA legacy overhead, no hardware specialisation |
| **Clock-aware Gen 2** *(custom silicon, declared tiers)* | Clock-aware CPU, HBM4 Task tier, LPDDR6 Session tier, no discrete GPU | ~HK$8,000 | Memory bandwidth (no other bottleneck — all eliminated) |
| **Clock-aware Gen 2 + GPU ALU array** | Gen 2 CPU + specialised matmul/softmax silicon (no CUDA generality) | ~HK$5,000 | Pure compute throughput — all other overhead eliminated |

The Gen 1 estimate reflects running the clock-aware software model on existing commodity hardware: no custom silicon, but the runtime eliminates OS overhead, removes the scheduler jitter, pre-allocates memory, and eliminates GC. The performance improvement comes purely from software — the hardware is unchanged. The cost reduction comes from not needing discrete GPUs, because the unified memory and the optimised runtime together bring inference within reach on a standard server.

The Gen 2 estimate reflects what happens when the hardware is redesigned for the model: OOO engine eliminated (→ more cores), L3 eliminated (→ more execution ports), coherency hardware eliminated (→ simpler die), VRAM/DRAM boundary eliminated (→ one pool). The 80–120 clock-aware cores per die replace 16 OOO cores, at lower manufacturing cost per core and better yield. The HBM4 tier is sized only for the declared `Task`-scoped working set — a fraction of what a GPU's VRAM pool requires.

The Gen 2 + GPU ALU array estimate reflects replacing CUDA cores with operation-class-specialised units: the matmul array, the softmax reduction tree, the cordic pipeline. No generalisation tax. Every transistor in the accelerator does exactly the operation the declared workload requires.

To put this in concrete terms: **Gen 2 brings the highest-quality AI models within reach as local agents for ~HK$5,000**. A 70B parameter model running on a Gen 2 clock-aware machine — with HBM4 for the declared working set, LPDDR6 for session state, and a specialised matmul array — runs at production serving throughput on hardware that costs less than a high-end consumer laptop today. Not in a data centre. Not behind an API. On a machine you own, with a signed manifest that proves what it is running, at what timing, under what compliance constraints. The atom stream is the audit log. The cryptographic proof chain is the identity. The compiler is the gatekeeper. The model runs locally, verifiably, and continuously — without a cloud subscription, without data leaving the machine, and without trusting anyone's infrastructure except the silicon you can physically inspect.

The broader consequence is the democratisation of serious compute. Today, running a frontier AI model in production requires either a cloud budget measured in thousands of dollars per month or a capital investment in GPU infrastructure measured in hundreds of thousands. Both gatekeep who can build production-grade AI systems. Gen 2 removes that gate: a solo developer, a small team, a researcher with limited funds can own hardware that runs the highest-quality models locally, continuously, at production throughput. An idea that today requires a data centre contract can be put to life on a machine that costs less than a month of GPU cloud spend. The cost of turning an idea into a running system drops by an order of magnitude — and the system that runs it is more verifiable, more auditable, and more correct than anything running in the cloud today.

These are speculative estimates, not product specifications. The derivation is: remove each hardware component that exists only to compensate for undeclared timing, replace it with computation, count the transistors saved, and price them at current fab cost per mm². The direction is not speculative. The magnitude is.

There is a further compounding dynamic that does not appear in these numbers: **the clock-aware processor manufactures its own successor**. A Gen 1 machine running frontier models locally generates atom streams — continuous, cycle-accurate records of every operation, every memory access, every declared workload. Those atom streams are the highest-quality dataset that has ever existed for silicon design: not benchmarks, not synthetic traces, but real execution at production throughput on real workloads with ground-truth timing. The ML substrate described in Paper III closes the loop — the atom stream trains the models, the models produce directives, the compiler incorporates them — but the same loop applies to the chip itself. Chip designers using Gen 1 hardware to run EDA tools, placement algorithms, and timing analysis on atom stream data from the previous generation will produce Gen 2 silicon that is cheaper, more efficient, and more precisely fitted to declared workloads than anything designed against synthetic benchmarks today. Gen 2 produces Gen 3 at lower cost still. Each generation's atom stream is a more complete characterisation of the workload space than any prior generation's benchmarks. The cost curve does not flatten — it accelerates downward. The ~HK$5,000 figure for Gen 2 is not a floor; it is a ceiling on what the first generation of this feedback loop will achieve.

The feedback loop extends into the fab itself. A clock-aware processor produces an atom stream from every operation it executes, including EDA workloads, timing analysis, and placement runs. Those streams characterise, at cycle resolution, exactly how the workload exercises the silicon. That same resolution — applied to the manufacturing process — enables something that no current fab does systematically: **running ML live during fabrication to correct defects as they form**. Today, yield improvement is a statistical process: expose a wafer, measure the batch, adjust the recipe for the next run. The defect and the correction are separated by an entire production cycle. With a clock-aware substrate driving the fab's own metrology circuits — etch depth sensors, alignment cameras, thermal monitors, each declared as `ReadOnlyDevice<T>` channels — the atom stream captures every measurement at every deposition and etch step. An ML model trained on prior atom streams predicts which sites are drifting toward a defect class before the defect is complete, and the fab's actuator circuits — laser power, gas flow, chuck temperature — receive a corrective directive within the same declared clock window. The defect is corrected in the same run it is forming. Yield improves not batch-to-batch but wafer-to-wafer, then die-to-die. The same model that closes the software execution loop closes the hardware manufacturing loop. The atom stream is not just a dataset for designing the next chip; it is the control signal for building the current one correctly.

```
FAB PROCESS — LIVE ML CORRECTION LOOP
──────────────────────────────────────────────────────────────────────────────

  FABRICATION STEPS                    METROLOGY DEVICES (declared channels)
  ─────────────────                    ─────────────────────────────────────
  Deposition          ──────────────►  ReadOnlyDevice<EtchDepth>
  Etch                ──────────────►  ReadOnlyDevice<AlignmentDelta>
  Lithography         ──────────────►  ReadOnlyDevice<ThermalMap>
  CMP                 ──────────────►  ReadOnlyDevice<SurfaceUniformity>
                                                   │
                                                   │  atom stream
                                                   ▼
                                        ┌──────────────────┐
                                        │   ATOM STREAM    │
                                        │  (cycle-accurate │
                                        │  measurement log)│
                                        └────────┬─────────┘
                                                 │
                               ┌─────────────────▼──────────────────┐
                               │           ML MODEL                  │
                               │  trained on prior generation atoms  │
                               │  predicts: site drifting toward     │
                               │  defect class? → severity estimate  │
                               └─────────────────┬──────────────────┘
                                                 │  directive (same clock window)
                                                 ▼
                                   ACTUATOR CIRCUITS (declared channels)
                                   ─────────────────────────────────────
                                   WriteOnlyDevice<LaserPower>
                                   WriteOnlyDevice<GasFlow>
                                   WriteOnlyDevice<ChuckTemperature>
                                                 │
                                                 ▼
                                      CORRECTION APPLIED
                                   (within the same deposition
                                    or etch step — not next batch)

  TODAY (statistical)          │   CLOCK-AWARE (continuous)
  ─────────────────────────    │   ─────────────────────────────────
  Expose wafer                 │   Measure at every step (atom stream)
  Measure batch after          │   ML predicts drift before defect forms
  Adjust recipe for next run   │   Actuator corrects within same window
  Defect ↔ correction: 1 run  │   Defect ↔ correction: 1 clock cycle
  Yield improves batch-to-batch│   Yield improves die-to-die

                    ┌─────────────────────────────────────┐
                    │  INTER-GENERATION FEEDBACK          │
                    │                                     │
                    │  Gen N atom streams                 │
                    │    → train EDA + fab ML models      │
                    │    → Gen N+1 chip designed cheaper  │
                    │    → Gen N+1 fab runs cleaner yield │
                    │    → Gen N+1 atom streams richer    │
                    │    → repeat                         │
                    │                                     │
                    │  Cost curve: accelerates downward   │
                    └─────────────────────────────────────┘
```

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

## The GPU as a Mathematical Circuit Array

A conventional GPU exposes a homogeneous grid of CUDA cores — each core capable of any floating-point operation. The homogeneity is a design choice driven by the inability to predict, at hardware design time, which mathematical operations the programmer will need. The result is a general-purpose compute array that is efficient at parallel floating-point but structurally identical for every operation type.

The clock-aware model inverts this. Because the compiler knows at compile time which mathematical operations a GPU circuit will execute — and at what volume, frequency, and precision — the hardware can be specialised per operation class rather than homogenised:

| Operation class | Hardware unit | Optimised for |
|---|---|---|
| Linear algebra (matmul, dot) | Dense multiply-accumulate array | Throughput — maximum FLOPs per mm² |
| Exponential / logarithmic | Approximation pipeline (Taylor, range reduction) | Latency — fixed cycles regardless of input |
| Geometric (sin, cos, atan2) | Cordic pipeline | Accuracy — bit-exact at declared precision |
| Softmax / normalisation | Reduction tree + reciprocal unit | Parallelism — vector-width normalisation in one pass |
| Integer / bitwise | ALU array | Zero overhead for quantised inference |

The compiler, knowing the operation mix of each GPU circuit from its declared channels and source code, emits a resource allocation that routes each operation to the correct unit. The GPU die is partitioned by operation class, sized by the declared workload, not by worst-case generalisation. A transformer inference circuit that is 90% matmul and 10% softmax gets a die allocation of 90% dense multiply-accumulate and 10% reduction tree — not 100% CUDA cores doing both inefficiently.

The consequence is that inference throughput per mm² increases not because the transistors are faster but because every transistor is doing the right operation for its circuit. CUDA cores are a generalisation tax. Remove the tax, and the die area directly becomes computation.

### Channel Is Broadcast — One Signal to All Subscribed Devices

A `Channel<T>` write in the clock-aware model is not a point-to-point message. It is a **broadcast**: a single write to the channel simultaneously delivers the value to every circuit subscribed to that channel — including circuits on other cores, circuits backed by different memory tiers, and circuits that are hardware peripherals rather than software circuits.

This means a `Channel<SensorReading>` produced by a sensor driver is received in the same tick by every circuit that declared a subscription to it — a display circuit, a logging circuit, an ML inference circuit, a control loop circuit — without the producer knowing or caring how many consumers exist. The compiler resolves the subscriber list from `system.cap` at build time and emits the correct fanout pattern for the physical topology (shared cache line, inter-core message, DMA ring).

From the programmer's perspective, a channel write is one operation. From the hardware's perspective, it is a single cache line write that the cache coherency mesh fans out to all declared subscribers simultaneously. The broadcast is structural — not a runtime pub/sub dispatch, not an event bus, not a message broker. One write. All subscribers. Declared at compile time. Zero runtime overhead.

### Precise and Imprecise Resources

Not all hardware resources behave deterministically. The clock-aware model distinguishes two classes:

**Precise resources** — resources whose timing is fully deterministic and compiler-provable. Every execution unit in the CPU, every DMA controller, every SRAM bank is a precise resource. The compiler assigns precise resources in the dispatch table with cycle-accurate bounds. A precise resource that deviates from its declared timing is a hardware fault, detected by the trace unit.

**Imprecise resources** — resources whose timing has a distribution rather than a fixed value. A DRAM row activation, a NVMe read with queue depth contention, a network round-trip — these have declared *bounds* but not declared *exact cycles*. The compiler treats imprecise resources as `Cold` accesses: the declared bound is verified as achievable in the worst case, the actual cycle is whatever the hardware delivers, and the circuit's declared window covers the bound. The gap between actual and bound is absorbed as slack.

The distinction matters for correctness. A `Permanent` tier access (DRAM, pinned) is precise — the compiler knows the exact row-to-column latency. A `Cold` tier access (NVMe) is imprecise — the compiler knows the declared worst-case, but not the actual. A circuit that treats an imprecise resource as precise — declaring a cycle budget too tight for the worst case — is a compile error. The compiler, reading the resource classification from `system.cap`, rejects declarations that conflate the two.

### Device Learning and Calibration

The clock-aware model can prove when a hardware device is not meeting its declared specification. Every device in `system.cap` declares its timing characteristics — the CPU's execution unit latencies, the DRAM's row activation time, the NVMe's read latency distribution. The trace unit continuously measures actual device behaviour against these declarations.

When the measured latency of a precise resource deviates from its declared value, the `ObservabilityCircuit` emits a `Channel<DeviceViolation>` signal: the device, the declared latency, the measured latency, and the tick at which the deviation occurred. This is not a performance alert. It is a correctness proof failure — the hardware is not behaving as the compiler assumed.

The uses of this are precise:

1. **Manufacturing defects.** A CPU core that consistently delivers 10% more latency than its speed grade declares is a binning error. The trace proves it. The runtime can migrate circuits away from that core automatically, via `Channel<CoreEviction>`, and flag the device for replacement.

2. **Thermal degradation.** A device that meets spec at cold but drifts under sustained load has its drift characterised by the atom stream over time. The compiler can be re-fed the measured latencies as a new `system.cap` entry — the calibrated model — and recompile against the device's actual behaviour rather than its nominal spec.

3. **Workload validation.** A system delivered to a customer under a timing SLA can prove, from the continuous trace, that every declared SLA was honoured in every production tick. Not a benchmark. Not a load test. A continuous, hardware-sourced, cryptographically-signable proof of execution.

The device is not just a declared resource. It is a continuously measured, self-calibrating input to the compiler model. The compiler's assumptions are always verifiable against reality, and the system adapts when reality diverges.

---

## Relation to ADR-0009

ADR-0009 (The Implied Hardware Architecture) in this repository records the same observation in decision form: that the clock-aware software model implies a hardware architecture, and that the compounding value of the primitive lives in the hardware convergence rather than in the software optimisation alone. This paper expands that observation into a full analysis.

The current implementation targets commodity x86-64 and AArch64 hardware — the near-term path is practical and does not require custom silicon. ADR-0009 and this paper describe the endpoint the model implies, not the starting point the implementation requires.

---

*Part of the clock-aware programming series. See [Paper I: Clock-Aware Programming](01-clock-aware-programming.md) for the core primitive. See [Paper II: The Language](02-language.md) and [Paper III: The OS and Runtime](03-os-and-runtime.md) for the software stack consequences.*
