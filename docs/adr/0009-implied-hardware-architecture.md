# ADR-0009: The Implied Hardware Architecture

**Status:** Speculative — a logical consequence of the model, not a current implementation decision.  
**Date:** 2026-06-08

---

## Context

Clock-aware programming is designed to run on existing commodity CPUs. The current CPU microarchitecture — out-of-order execution, speculative execution, branch prediction, a deep cache hierarchy (L1/L2/L3) — was designed for a world where software never declares its timing. The CPU cannot trust the instruction stream to be optimally ordered, so it built a runtime prediction machine to discover the ordering the compiler never provided. That machine — the ROB, reservation stations, branch predictor, prefetch engine, store buffer, TLB — consumes a significant fraction of die area and power.

Clock-aware programming changes the information available at compile time. The compiler knows the schedule. The logical question is: what does a CPU designed for a clock-aware software model look like, if hardware engineers could design for it directly?

This ADR records that question and its answer — not as a current implementation decision, but as the implied hardware destination of the model, and the reason the software primitive has compounding value as transistor density increases.

---

## Decision

The clock-aware programming model implies, as its logical hardware endpoint, a CPU that:

1. **Executes in-order**, trusting the compiler-declared schedule rather than recovering it at runtime.
2. **Has no branch predictor** — branch-free hot paths are a declared precondition; nothing to predict.
3. **Has no out-of-order machinery** — no ROB, no reservation stations, no reorder buffer. The compiler already scheduled optimally from the declared dependency graph.
4. **Has no speculative execution** — nothing to speculate about when the schedule is known.
5. **Has no L2 or L3 cache for annotated working sets** — the working set is declared and pinned in L1. L2 and L3 exist to absorb misses from *unpredictable* access patterns; with declared regular patterns and L1-pinned data, the miss hierarchy is idle on annotated paths.
6. **Has a strong memory model (TSO or stronger)** — enabling full RCU elimination without residual barriers, as the cycle-window proof holds end-to-end.
7. **Reinvests freed die area** into more execution ports, wider SIMD units, more cores, and higher clock frequency.

---

## Rationale

### OOO is expensive and still non-deterministic

Out-of-order execution is the most costly subsystem on a modern CPU die — in area, power, and validation complexity — and it still does not deliver deterministic timing. This is the fundamental contradiction at the heart of the current architecture.

OOO costs: a reorder buffer with hundreds of entries, reservation stations, a complex instruction scheduler, register renaming logic, speculative load/store queues, and the full Spectre/Meltdown attack surface and its mitigations. At advanced process nodes this machinery consumes a significant fraction of die area and a disproportionate share of dynamic power — the OOO engine operates on every instruction, every cycle, regardless of whether reordering is beneficial.

What OOO delivers in return: higher average throughput for irregular workloads by hiding memory latency and exploiting instruction-level parallelism. For general-purpose compute where the access pattern is unpredictable and the instruction stream is irregular, this is a genuine win.

What OOO does not deliver: deterministic timing. The actual execution latency of any given instruction sequence on an OOO CPU is a distribution, not a point. The reorder buffer depth, the instruction mix, the cache state, the branch predictor history, and the presence of other threads on the same physical core all affect when a given instruction completes. This jitter — tens to hundreds of nanoseconds on a modern server CPU — is not acceptable in HFT, audio processing, safety-critical control systems, modem frame scheduling, or any field where a timing deadline is a hard requirement rather than a statistical target.

The consequence is that OOO hardware forces two classes of workarounds: software workarounds (the entire apparatus of `isolcpus`, `nohz_full`, `irqaffinity`, pinned frequency, DPDK poll mode — all the preconditions in ADR-0002) to suppress the jitter sources, and statistical workarounds (p99/p999 latency targets, worst-case execution time analysis, timing slack budgets) to tolerate the jitter that remains. Both workaround classes have real cost: the software workarounds require dedicated cores and careful configuration; the statistical workarounds require over-provisioning cycle budgets to accommodate the tail distribution.

A clock-aware in-order CPU eliminates both workaround classes simultaneously. The jitter source is gone — not suppressed, not tolerated with margin, gone — because in-order execution with a declared schedule produces the same cycle count on every run. The tail distribution collapses to a point. The p99 equals the p1. The worst-case execution time is the only execution time.

This is what "expensive and still non-deterministic" resolves to: OOO is the wrong tradeoff for any workload that has a timing requirement, and a growing fraction of computing workloads — real-time, safety-critical, latency-sensitive, power-constrained — has timing requirements. Clock-aware in-order hardware is cheaper to build, consumes less power, and delivers the guarantee that OOO cannot.

### Simpler schematics reduce manufacturing cost and improve yield

Chip cost is not purely a function of die area — it is a function of die area, defect density, and design complexity. A simpler circuit has fewer transistors, shorter interconnects, fewer clock domains, and fewer timing-critical paths. Each of these reduces manufacturing cost through three mechanisms:

**Smaller die area.** Fewer transistors means smaller die. A smaller die fits more units per wafer. More units per wafer means lower cost per unit at the same defect rate. The freed area from eliminating OOO, branch prediction, L2/L3, and speculative execution hardware directly reduces die size — and a smaller die compounds with every process node because the absolute reduction in area translates to proportionally more units per wafer.

**Higher yield.** Yield — the fraction of dies on a wafer that pass functional testing — falls with die area. Larger dies intersect more random defects. A clock-aware CPU's smaller die area means fewer defect intersections per unit, directly raising yield. At advanced process nodes (3nm, 2nm) where defect density is higher and yield is a primary cost driver, this is significant.

**Simpler validation.** The OOO engine, branch predictor, and speculative execution machinery are among the hardest blocks to verify correctly — they are the source of most post-release errata (Spectre, Meltdown, and every speculative execution vulnerability since are consequences of this complexity). Simpler circuits have fewer corner cases, shorter validation cycles, and lower risk of post-silicon bugs. Validation cost is a real and large fraction of total chip development cost; reducing it is a direct saving that does not appear on the die area budget but is real nonetheless.

The manufacturing cost reduction is not a secondary benefit — it is structural. A clock-aware CPU is cheaper to build per unit, yields more dies per wafer, and requires less post-silicon validation. At volume, those three factors compound into a significant cost advantage over an equivalently-noded OOO design. The software primitive that eliminates runtime complexity in the kernel also eliminates hardware complexity in the factory.

### Fully deterministic access = prefetcher always correct

The hardware prefetcher works by detecting stride patterns in memory accesses and issuing loads ahead of demand. On a general-purpose OOO CPU it is a predictor — it is correct when the access pattern is regular and wrong when it is not. Mispredictions cause pipeline stalls that the OOO engine then attempts to hide with speculation.

On a clock-aware CPU with declared, regular, L1-pinned working sets, the prefetcher is not a predictor — it is a lookup. The access pattern is identical every iteration, declared at compile time, and never deviates. After one iteration the prefetcher has seen the complete pattern; after two it has confirmed it; from the third iteration onward it is correct 100% of the time, indefinitely. Not approximately correct, not statistically correct — exactly correct, because the schedule is deterministic and the pattern cannot change.

The consequence is that the prefetcher's miss penalty drops to zero on annotated paths. There are no mispredicted prefetches to recover from, no speculative loads to cancel, no pipeline stalls to hide. The prefetcher transitions from a probabilistic optimisation to a deterministic guarantee — and on a clock-aware CPU where the OOO engine is absent, that guarantee is the only mechanism needed. The prefetcher becomes trivially simple: detect the stride once, issue loads at the declared interval forever. It needs no history buffer, no confidence counter, no adaptive logic. Another piece of hardware that was complex because software was unpredictable collapses to a simple circuit when software declares what it will do.

### Every eliminated subsystem was compensating for unknown timing

The OOO engine exists because the instruction stream is not optimally ordered for the hardware — the CPU must discover the ordering at runtime. Tell the CPU the ordering at compile time and the OOO engine is dead silicon.

The branch predictor exists because the CPU cannot see which branch will be taken. A branch-free hot path has no branches to predict.

The L2 and L3 caches exist because access patterns are unpredictable — the CPU must buffer a large working set to absorb misses. A declared, regular, L1-pinned working set has no misses to absorb.

Speculative execution exists to hide latency from unpredictable operations. With declared timing, no operation has unpredictable latency.

Each eliminated subsystem follows the same logic as RCU, memory barriers, and the scheduler in software: it compensates for not knowing the schedule. Declare the schedule and the compensation becomes unnecessary.

### Freed die area compounds with transistor scaling

This is where the hardware implication becomes exponentially significant.

As transistors become smaller, the die area freed by removing the prediction machine does not simply disappear — it is reinvested. Every process node (5nm → 3nm → 2nm) makes transistors smaller and cheaper per unit area. The freed area from eliminating OOO, branch prediction, and the L2/L3 hierarchy translates, at each new process node, into:

- **More execution ports** — from 3–5 ports on a current OOO core to 8–16+ ports on an in-order core designed for a clock-aware schedule, because the compiler can fully exploit all ports without runtime arbitration.
- **More cores per die** — simpler core = smaller die area per core = more cores in the same package.
- **Higher clock frequency** — simpler pipeline = fewer pipeline stages = lower critical path delay = higher achievable frequency within the same thermal envelope.
- **Lower power per operation** — eliminated speculation and prediction logic consumes no power. More thermal headroom available for computation.

The compounding is structural: each process node amplifies the gain from the simpler architecture. An OOO CPU at 3nm is faster than an OOO CPU at 5nm by some factor. A clock-aware in-order CPU at 3nm is faster by a larger factor, because it reinvests more of the node's transistor budget into actual computation rather than prediction machinery.

The FPGA is the existence proof. FPGAs are in-order by construction — every operation executes exactly when declared. As FPGA process nodes have shrunk, the gains have gone entirely into more logic elements, wider datapaths, and higher clock rates, not into prediction hardware. A clock-aware CPU follows the same trajectory.

### The memory model becomes simple and strong

Without OOO and speculative loads, the memory model simplifies to TSO or stronger naturally. Stores complete in program order. Loads observe stores in program order. The store buffer exists only to absorb write bandwidth, not to reorder or speculate. The RCU cycle-window proof holds end-to-end without residual barriers — the precondition that enabled only partial elimination on existing OOO hardware becomes fully satisfied.

### Hardware frequency exposure enables runtime self-calibration

On a clock-aware CPU, the hardware exposes its actual operating frequency as a stable, readable value at boot — not a nominal specification, but the frequency the silicon is actually running at on this unit, at this temperature, on this power rail. The runtime reads this value once at startup and calibrates all `budget_ns` annotations to exact cycle counts from the actual frequency rather than the declared estimate in `system.cap`.

This closes the last calibration gap in the current model. Today, `cpu_freq_mhz` in `system.cap` is a static declaration that the programmer must pin externally (`cpupower frequency-set`) and verify holds at runtime. On a clock-aware CPU, the hardware guarantees the frequency and reports it directly. The software reads it, converts `budget_ns` to cycles, and the verification is exact — not approximate. The `cpu_freq_mhz` field in `system.cap` becomes a compile-time lower bound for static verification; the runtime value from the hardware is the authoritative source for cycle budget computation.

The practical consequence: a clock-aware binary is self-calibrating. It does not need to be rebuilt for a 3.6 GHz part versus a 4.2 GHz part — it reads the hardware frequency at boot and adjusts. The compile-time proof confirms the budget is achievable at the declared minimum frequency; the runtime confirms the actual frequency and sets the exact cycle window.

This generalises into a continuous feedback loop, particularly valuable on mobile. A mobile CPU operates across a wide frequency range: peak frequency under load, reduced frequency under thermal or battery pressure. Today, the OS power governor manages this heuristically — it raises and lowers frequency based on utilisation measurements that lag actual demand. Under a clock-aware model, the feedback loop is closed precisely: the hardware reports its current frequency at each cycle boundary; the runtime recomputes cycle budgets from the actual frequency; circuits self-regulate their poll intervals and execution windows to match. If the CPU throttles from 3.2 GHz to 2.4 GHz under battery pressure, the runtime detects it immediately at the next boundary and widens cycle windows proportionally — not by guessing from a utilisation heuristic, but from the exact frequency the hardware reports. The declared timing budget in nanoseconds remains valid across the frequency transition; only the cycle count it maps to changes, and the runtime adjusts that mapping automatically. The result is a system that adapts to hardware state changes with zero latency and zero overshoot — a control loop whose feedback signal is the hardware itself.

This also benefits a large class of existing software that already expresses timing as a multiple of the system frequency: audio sample rates, video frame timing, network protocol timeouts, sensor polling intervals, game loop tick rates. All of these currently discover the base clock indirectly — through `clock_gettime`, `QueryPerformanceFrequency`, TSC calibration loops, or hardcoded assumptions that break silently when the CPU throttles. A clock-aware CPU that exposes its actual operating frequency as a first-class hardware value closes this loop for all of them simultaneously. The runtime reads it once; every frequency-relative timing computation in the system gets the correct, current answer — not an approximation from a calibration heuristic, not a stale value from before a throttle event.

### Benchmarks become exact

A secondary but significant consequence: benchmarks become structurally accurate rather than statistically approximated.

Today, benchmarking a piece of code requires multiple runs, warm-up phases, outlier filtering, and careful control of CPU frequency (`cpupower`, `GOARCH`, pinning to a single core) to get a stable result. Even then, the result is a distribution — median, p99, p999 — because OOO execution, branch mispredictions, cache cold-starts, and frequency transitions all introduce variance that no amount of statistical aggregation fully eliminates. A benchmark measures the *average behaviour of a probabilistic system*, not the *actual cost of a declared operation*.

On a clock-aware CPU with declared schedules and hardware-exposed frequency, a benchmark of an annotated path produces the same result on every run. The schedule is deterministic. The frequency is known. The working set is pinned. There is no variance to average over — the first run and the thousandth run produce identical cycle counts. The benchmark stops being a statistical exercise and becomes a direct measurement: the compiler's declared cost, confirmed by the hardware executing exactly what was declared.

This means performance regressions are detectable from a single run. CI pipelines that today require hundreds of benchmark samples to detect a 2% regression with statistical confidence would detect any regression — exactly — from one. The measurement infrastructure that exists to fight benchmark noise becomes unnecessary, for the same reason the runtime synchronisation infrastructure becomes unnecessary: the source of the noise was unknown timing, and declaring the timing eliminates the noise.

This also closes the gap between simulation-based profilers and reality. Cachegrind, for example, works by simulating a cache model against the instruction stream — it produces a perfect count of cache events *in its model*, but that model diverges from the real hardware because OOO reordering changes the actual access order at runtime in ways the simulation cannot predict. On a clock-aware CPU, the actual access order is the declared order: in-order execution means the instruction stream the simulator sees is the instruction stream the hardware executes. The working set is pinned, the access pattern is regular, and the hardware cache behaviour matches the simulation model exactly. Cachegrind's output stops being an approximation and becomes a direct measurement — the simulated miss count and the actual miss count converge to the same number. Profiling tools that today require hardware performance counters to correct for simulation inaccuracy become unnecessary; the simulation is the reality.


### The cache hierarchy collapses for annotated paths

For a declared working set pinned in L1, L2 and L3 serve no purpose on the annotated path. The freed die area — L2 and L3 together often consume 50–70% of a modern CPU die — is available for computation. In the context of a chip designed from scratch for clock-aware software, that area budget is the largest single contributor to the reinvestment gain.

---

## Consequences

- **Full RCU elimination** — no residual barriers required on any memory model, because the hardware model is strong by construction.
- **Full memory ordering elimination** — the compiler schedule is the memory order; no `smp_mb()`, no `std::atomic`, no `READ_ONCE` required anywhere.
- **Verification model simplifies** — `llvm-mca` becomes exact rather than approximate; the CPU executes precisely the instruction schedule the compiler produced, with no runtime deviation.
- **The margin required in `budget_ns` annotations shrinks to near-zero** — current margin compensates for OOO variance; without OOO, the only variance is manufacturing tolerance at a given process node, which is small and well-characterised.
- **The simplification thesis closes** — every piece of compensating machinery in software was eliminated because the schedule was declared; every piece of compensating machinery in hardware is eliminated for the same reason.

---

## Relationship to Current Work

This ADR describes the logical endpoint, not the current implementation. Clock-aware programming on existing OOO x86 hardware is the correct starting point:

- It works today on commodity hardware.
- It produces real, measurable gains (RCU reduction, barrier reduction, scheduler bypass) without requiring new silicon.
- It creates the body of clock-aware annotated software and toolchain infrastructure that a future clock-aware CPU would consume.

The path is: annotate software → prove the model works → accumulate the annotated codebase → build hardware optimised for that codebase. This is the same path RISC took against CISC: software first proved that simpler instruction execution was sufficient; hardware then optimised for that simpler model. The hardware win came after the software proof, not before.

---

## Alternatives Considered

**Continue optimising OOO hardware** — deeper ROBs, larger predictors, more speculation. This is the current industry trajectory. It continues to improve performance but the gains are diminishing: each process node delivers smaller improvements to OOO prediction accuracy than to raw transistor density, because prediction is a fundamentally bounded problem. Clock-aware hardware escapes that bound.

**VLIW (Very Long Instruction Word)** — a prior attempt at compiler-scheduled in-order execution. VLIW requires the compiler to schedule across the full instruction word, which creates binary compatibility problems when the hardware changes. Clock-aware in-order execution is not VLIW: the scheduler is the timeslice checker, not the instruction packager, and binary compatibility is maintained through `cpu_model` in `system.cap`.

**DSPs and fixed-function accelerators** — in-order by construction, efficient for declared workloads. Clock-aware CPUs generalise this: a programmable core that is in-order by construction when the software declares its schedule, and falls back to conventional execution for unannotated paths.
