## Execution

### The Kernel Does the Hardware's Job — Better

Every optimisation a modern CPU performs speculatively, the clock-aware kernel performs provably.

Out-of-order execution exists because the hardware cannot know which instructions are independent — so it guesses, renames registers, and reorders speculatively, paying the cost of a reorder buffer and a misprediction flush. The compiler knows the full declared instruction sequence before any execution happens. It reorders statically, at build time, with no speculation and no flush penalty. The OOO engine is not needed — the compiler already did its job, correctly, once, before the binary shipped.

Branch prediction exists because the hardware cannot know which branch a runtime value will take. The compiler eliminates branches on the hot path entirely: truth tables, `cmov`, exhaustive match compiled to jump tables. What the branch predictor guesses at, the compiler proves. The predictor is not needed for what was already decided at compile time.

Register renaming exists because the hardware cannot know the programmer's intent — it must infer independence from the instruction stream. The runtime's register liveness map is derived from the manifest's declared channel types. The compiler knows exactly which registers hold live channel values across window boundaries. It names them directly and permanently. No renaming needed — the names were declared.

The prefetcher exists because the hardware cannot know the access pattern ahead of time — it detects strides after the fact and speculates forward. The compiler knows every memory access the next circuit will make before that circuit's window opens, because the access pattern is declared in the manifest. The runtime issues exact prefetches during idle ticks. The prefetcher is not needed — the pattern was already known.

Memory barriers exist because the hardware cannot know the programmer's ordering intent — it must be told explicitly when to stop reordering. The compiler proves ordering at window boundaries (Gathering) and via hardware signals (Non-Gathering). The ordering is a theorem. No barrier instruction is needed to assert what was already proved.

The cache hierarchy — L1, L2, L3 — exists because the hardware cannot know the working set. It speculates with LRU eviction and hardware prefetch. The compiler declares every circuit's working set in the manifest and pins it to the correct tier before the circuit's first instruction. The cache does not need to discover the working set — it was told.

**The pattern is always the same.** The hardware performs a function speculatively, generically, at runtime, with a statistical success rate, paying silicon area and power for the mechanism. The clock-aware kernel performs the same function provably, specifically, at compile time, with a 100% success rate, paying nothing at runtime. The hardware's mechanisms exist because the software never provided the information that would make them unnecessary. Declare the information, and the mechanisms dissolve — not because they are replaced by something better, but because the problem they were solving no longer exists.

This is not an incremental optimisation. It is a transfer of responsibility from hardware to software — from the silicon's speculative engines to the compiler's proof system. The hardware that remains after this transfer has no wasted transistors. Every gate computes. Every cycle is declared. Every outcome was already known.

---

### The Runtime Is the Scheduler

The scheduler exists to resolve contention between tasks whose timing is unknown. In a clock-aware system, there is no such contention — every function's window is declared non-overlapping. There is nothing to arbitrate. The "scheduler" in this model is the clock itself: functions execute at their declared cycles, in order, because the CPU executes the next instruction. No arbitration. No priority queue. No run queue. No preemption. The cycle annotation is the schedule; the hardware is the scheduler.

**The entire kernel is event-driven. The only event is a clock tick.**

Linux is driven by a mixture of events: timer IRQs, hardware IRQs, softirqs, syscalls, page faults, IPIs, SMIs. Each has its own entry path, its own latency, its own interaction with the scheduler. A task has no way of knowing which of these fired while it was nominally running, for how long, or how many times. When a Linux task calls `RDTSC` before and after a block of code, the delta includes all of it — scheduler preemption, IRQ handlers, TLB shootdowns, firmware SMIs — with no labels, no attribution, no way to recover the breakdown. The task knows it yielded when it called `schedule()`. At all other times, it has no knowledge of whether it was actually running.

The clock-aware kernel has one event type: a `CLKIN` tick. That tick advances the dispatch table by one entry. The circuit whose window opens at that tick executes. There is no IRQ classification, no softirq backlog, no preemption decision, no run queue traversal. The tick fires; the dispatch table determines what runs; the circuit runs to its declared window boundary; the next tick fires. Every CPU cycle is attributed to exactly one declared circuit. There are no unaccounted cycles. A task cannot be preempted without its knowledge because preemption does not exist — a circuit runs its full declared window, atomically, from the perspective of the dispatch table. The atom stream in the `ObservabilityCircuit` is the proof: every tick on every core is labelled with the circuit that owned it. No tick is anonymous. No tick is lost.

**IRQs become channel writes, not control-flow transfers.** In a conventional kernel, a hardware interrupt preempts whatever is running, transfers control to an interrupt handler, executes the handler, then returns — stealing an unknown number of cycles from the interrupted task with no attribution. The task had no say. It did not know. In the clock-aware model, the hardware signal — a NIC DREQ, a timer assertion, a device ready line — writes to a declared channel via DMA. It does not preempt anything. Control never transfers. The CPU does not jump to a handler. The interrupt becomes data at a physical address, and the dispatch table delivers it to the correct circuit — the networking stack, the timer circuit, the device driver — at the correct declared tick, in priority order.

Priority is expressed through two orthogonal declarations in `@Timeslice`, not through interrupt priority levels:

**Window position within an epoch** — which circuit fires first when multiple circuits share the same epoch. `NicCircuit` is assigned tick 0; `ObservabilityCircuit` is assigned a later position. When both happen to be due at the same epoch boundary, `NicCircuit` executes first by construction.

**Epoch period** — how often a circuit's window repeats. This is the dominant mechanism for responsiveness. `NicCircuit` declares a short period (e.g. every 4800 ticks at 4 GHz = 1.2 µs, matching the 10 Gbps frame arrival rate). `ObservabilityCircuit` declares a long period (300 million ticks = 100 ms). The dispatch table repeats each circuit at its declared interval:

```
tick        0: NicCircuit          (period: 4800 ticks)
tick        6: parsePrice          (period: 4800 ticks)
tick     4800: NicCircuit          (repeats)
tick     4806: parsePrice          (repeats)
...
tick 36_000_000: ObservabilityCircuit  (fires once per 100 ms)
```

When a network packet arrives via DMA at tick 2400, it writes to `channel EthernetFrame` and waits. The maximum time before `NicCircuit` picks it up is one `NicCircuit` period — 4800 ticks, 1.2 µs. The packet does not wait for `ObservabilityCircuit`. It does not wait for anything with a longer period. The worst-case latency for any event is bounded by the epoch period of the circuit that consumes it — a compile-time constant declared in `@Timeslice`, proved to fit in the dispatch table, known before the first packet arrives.

This is what replaces ARM GIC priority levels and Linux `IRQL`. In a conventional kernel, interrupt priority is a runtime arbitration mechanism — when two IRQs fire simultaneously, the higher-priority one runs first. In the clock-aware model there is no simultaneous arbitration because there is no preemption. The packet data is in the DMA buffer at its declared physical address; it is not going anywhere. `NicCircuit`'s next window is already in the dispatch table at a known tick. The maximum wait is the epoch period. The epoch period was declared by the programmer and proved by the compiler. The concept of "serving the network packet first" is expressed as declaring `NicCircuit` with a short period — not by elevating its interrupt priority at runtime.

**The runtime execution plan is not computed — it is read from the channel graph.** A general-purpose scheduler is complex because it must solve a resource allocation problem at runtime, without knowing task durations, without knowing data dependencies, without knowing which tasks will block and for how long. Every heuristic it uses — priority queues, CFS fairness weights, deadline-based ordering — is a compensating mechanism for not having that information.

The clock-aware runtime has all of that information, statically, before the first instruction executes. The channel graph is the data dependency graph. It is a compile-time constant. The runtime does not need to discover which circuit depends on which — that is declared in the subscriptions and proved in Pass 1. The execution plan is therefore not an NP-hard bin-packing problem solved at runtime. It is a topological sort of the channel graph, performed once at compile time, stored in the dispatch table, and read back by the runtime one entry at a time.

The only genuine scheduling decision the runtime faces is **arbitration between independent chains competing for application cores**. Two unrelated workloads — a trading pipeline and a risk aggregation pipeline — have no channel connections between them. The channel graph gives no ordering between them. This is the one case where a weighting mechanism is needed, and it is the simplest possible one: `@Timeslice` period is the weight. The shorter the period, the more frequently the chain head fires, the more app core time it consumes. A chain with a 1.2 µs period and a chain with a 100 ms period coexist on the same set of cores without interference because they occupy completely different tick ranges in the dispatch table. The compiler places them so their windows never overlap. When they do compete for the same window slot, the shorter-period chain takes precedence — it was declared to fire more urgently.

```
  Related circuits (channel graph gives order):
  NicCircuit → parsePrice → updateBook → emitQuote
  Ordering: topological sort of channel graph, compile-time, static

  Unrelated circuits (no channel connection — weighting arbitrates):
  TradingPipeline     (period: 1.2 µs)  ──► short period = high weight
  RiskAggregation     (period: 10 ms)   ──► long period  = low weight
  Arbitration: compiler places windows in non-overlapping slots;
               shorter period takes precedence at any contended boundary
```

The runtime's dispatch loop is therefore simple: follow the channel graph for dependent chains, use period-as-weight for independent chains, apply `early_fire` promotions for incoming signals. Three rules. No heuristics. No priority queues to maintain. No scheduler state to update. The entire execution plan was computed once, at compile time, by the compiler walking the channel graph. The runtime executes it.

**The unit of scheduling is the window, not the raw clock tick.** A CLKIN tick is one hardware clock cycle — 250 ps at 4 GHz. No function executes in a single cycle. What the runtime schedules is a *window*: a contiguous block of `budget_ticks` CLKIN cycles that the compiler has proved is sufficient to execute the circuit's entire function body, including pipeline depth. The budget is derived from the `@Timeslice` annotation:

```
@Timeslice(core = 1, period = "1ms")
circuit RiskCheck { ... }
```

The compiler walks the instruction graph of `RiskCheck`, counts cycles, adds pipeline drain depth, and asserts the total fits within `budget_ticks`. If it does not fit, it is a compile error. If it fits, the window is exact: every instruction executes within the budget, the pipeline drains, and the circuit returns. The window is atomic — it runs to completion or it does not run at all. The runtime never preempts it mid-window, because doing so would invalidate the timing proof for everything scheduled after it.

The "tick" in the scheduling shorthand "next tick" means "next period boundary" — the next time the circuit's declared `period` places its window in the dispatch table. A circuit with `period = "1ms"` has a window that recurs every million CLKIN cycles at 1 GHz. Between recurrences, the circuit is structurally absent. When the period boundary arrives, the runtime evaluates whether to execute the window or skip it. The raw clock is the ground for measurement; the period is the ground for scheduling.

**If a circuit cannot run at its period boundary, it waits for the next one.** There are exactly two reasons a circuit is skipped: its input channel is empty, or every core assigned to its clock model is occupied by another circuit's window. In either case the outcome is identical — the runtime does not execute the window at this period boundary. The core moves to the next dispatch table entry. There is no retry mechanism, no blocking call, no wait queue, no backpressure protocol to implement. At the next period boundary, the runtime evaluates both conditions again. If data has arrived and a core window is available, the circuit executes its full `budget_ticks` window. If either condition still holds, it is skipped again. The circuit is absent from execution until the wire has something on it and the silicon has a slot for it.

A saturated clock model is therefore first-class pressure that the runtime handles by the same mechanism as empty channels. The programmer declares nothing extra. There is no `async`, no `await`, no thread pool size to tune. The period boundary is the natural retry. Every circuit that was skipped automatically re-enters the dispatch table at its next declared period boundary, in topological order, and the same three rules apply. The system is always making progress on whatever can progress, without any scheduler policy beyond the compiled channel graph.

This is the simplest possible flow control — and it is correct by construction. There is no possibility of the circuit consuming stale data, because the channel is single-writer and the runtime only presents the circuit with the window when data is present. There is no possibility of busy-waiting, because the circuit generates zero instructions when absent. There is no possibility of dropped data, because the channel's declared size bounds how much can accumulate before the producer must slow down — and the compiler proves the producer's write rate is compatible with the consumer's read period. If they are not compatible, it is a compile error, not a runtime overflow.

**Network bursts are not a special case.** The obvious objection is: what happens when a burst of network packets arrives faster than the consuming circuit's window period? The answer is that the channel's declared size is the burst absorber — the NIC's DMA writes frames into the channel's ring buffer faster than the consumer drains them, and the ring buffer holds them until the next window. This is not a new problem requiring a new mechanism; it is what ring buffers have always done. The question is whether the declared size is large enough to absorb the burst without overflow. The compiler answers this at compile time: given the NIC's declared maximum burst rate (from `system.cap`) and the consumer circuit's declared window period, the minimum channel size to guarantee no loss is a theorem, not a tuning parameter. If the declared size is insufficient the programme does not compile.

The deeper point is one of scale. A 10 GbE NIC at line rate delivers one 1500-byte frame every 1.2 µs — 1,200 nanoseconds. A CPU window at 1 GHz is measured in single-digit nanoseconds. The scheduler and window machinery operate two to three orders of magnitude faster than the network can produce packets even under a burst. The channel is never at risk of being written faster than the CPU can read — the NIC is the slow device here. Every frame that arrives has hundreds of CPU windows available before the next frame could possibly land. The "burst overwhelms the window" scenario requires a network device that operates at CPU speed — which is not a network device, it is another CPU core, and that case is handled by the core's own channel subscription and dispatch table. The concern is real in a conventional OS because the scheduler introduces millisecond-scale latency that the NIC can fill. In a clock-aware system the window is nanosecond-scale and the NIC never catches up.

**No device ever outputs as fast as the scheduler runs — and this is why code looks sequential.** The NIC example generalises to every I/O device in the system. An NVMe SSD at peak sequential throughput delivers data every ~1–2 µs. A PCIe bus transaction completes in ~100–200 ns. A GPU DMA transfer initiates in ~1 µs. A sensor reading a SPI bus at 10 MHz produces one byte every 100 ns. In every case, the CPU window — nanoseconds — is faster than the device by at least one order of magnitude, and usually two or three.

This asymmetry is the reason the sequential programming model is not a lie. It *feels* correct to write:

```
val frame  = NicChannel.get(index)
val parsed = parse(frame)
val result = computeRisk(parsed)
RiskOutput.put(index, result)
```

as if these four operations happen one after another in time — because *they do*. Each circuit fires at its window boundary in topological order. `NicChannel` fills from the NIC DMA. `parse` fires next. `computeRisk` fires after `parse` writes. `RiskOutput` fires last. The channel graph is the sequence. The compiler proves the sequence is correct. The runtime executes it in order.

The reason conventional programming abandoned the sequential model — replacing it with callbacks, futures, async/await, actor queues, and thread pools — is that the scheduler could not guarantee the CPU would be available when the device was ready. The device could fill its buffer and overflow while the CPU was doing something else. The only safe response was to batch work, queue it, and process it asynchronously. The complexity of async programming is entirely a consequence of the scheduler being slower than the device — not slower in raw throughput, but slower in *latency to respond*. A scheduler that takes 1 ms to notice a packet has arrived has allowed 833 frames to accumulate at 10 GbE. Batching is not a choice in that model; it is a necessity.

Remove the latency gap and batching ceases to be necessary. A nanosecond-window scheduler that checks the NIC channel at every window boundary is present for every frame as it arrives. There is nothing to batch. The code is sequential because the execution genuinely is sequential — not because the programmer pretended it was and hoped the OS would not lie. The clock-aware model does not bring back sequential programming as an ergonomic convenience. It brings it back because it is *correct*.

**An IRQ signal promotes the consuming circuit to the top of the remaining dispatch table.** The epoch period is the worst-case latency bound — the maximum time a circuit waits if its channel receives data at the worst possible moment in its epoch. But a hardware signal that writes to a declared `channel IrqSignal` does more than deposit data. It also sets a single flag in the dispatch table entry for the consuming circuit: `early_fire = true`. The runtime's dispatch loop, at the close of the currently executing window — never mid-window, never preempting anything — reads that flag and inserts the signalled circuit's window immediately after the current one, ahead of whatever was next in the epoch. The circuit executes at the earliest clean window boundary after the signal arrived. Not at its normally scheduled epoch position. At the next available slot.

This is the exact semantic of a hardware IRQ, expressed without preemption. A conventional IRQ says: stop what you are doing and serve me now. A `channel IrqSignal` says: finish what you are doing, then serve me before anything else. The difference is the absence of mid-window preemption — the current circuit always completes its declared window, because its budget was proved to fit, because violating that proof would invalidate every timing guarantee in the system. The signal waits at most one window boundary. Then it is at the top of the table.

**IRQ response latency is now bounded by the CPU clock, not the OS scheduler.** The worst-case time between a signal arriving and the consuming circuit executing is `remaining_budget_ticks × CLKIN_period` — the ticks left in the current window multiplied by the period of one hardware clock tick. At 4 GHz that period is 250 ps. A circuit with a 12-tick budget has a worst-case IRQ response of 12 × 250 ps = 3 ns. In a conventional OS, the equivalent bound is measured in microseconds — the scheduler must save register state, classify the interrupt, run the handler, restore state — and that bound is not proved, it is measured and hoped to hold. The clock-aware bound is a theorem: `budget_ticks` is a compile-time constant, `CLKIN_period` is a silicon constant, and their product is the exact worst-case latency. Make the CPU faster and the latency shrinks at the same ratio, automatically, with no software change. No OS scheduler tuning. No kernel configuration. The silicon is the limit, and it always was — the OS was just the layer preventing you from reaching it.

### The Runtime Dispatch Loop — A Finite State Machine

The runtime's dispatch loop is a finite state machine. Four states repeat every declared clock window:

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   WATCH ──► PLAN ──► EXECUTE ──► EVALUATE ──► WATCH (repeat)   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**WATCH — Between windows.** The runtime's window closes and it becomes absent from execution — exactly like any other circuit with no pending data. It is subscribed to `channel ClockTick`. The hardware timer fires at the declared system tick rate, writes into `channel ClockTick`, and the runtime's next window opens. Between ticks the runtime generates no instructions and occupies no execution port. On cores with no circuit due this tick, the hardware is placed in `STANDBYWFI`, waking automatically on the next timer event. There is no spin. There is no idle loop. The runtime is absent between its own windows, just as application circuits are absent between theirs.

**PLAN — Next window identified.** An `early_fire` flag is set, or the counter has reached the next scheduled window boundary. The runtime reads the dispatch table entry. Which circuit fires this window — the normally scheduled one, or an `early_fire` promoted circuit? The plan is not computed at this point — it was compiled. The runtime reads a static structure, not a dynamic queue.

**EXECUTE — Circuits run.** The CPU executes the declared instruction sequence within the declared window. The runtime observes via the trace unit. It does not intervene.

**EVALUATE — Window closes.** The runtime reads the actual cycle count from the trace unit, computes the delta against the declared budget, and writes it to the observability channel. If the delta is consistently trending toward the budget ceiling, a pre-warning signal is emitted before any violation occurs. Clock model adjustments — frequency pre-conditioning for an upcoming declared burst — are applied here, ahead of the burst, using the dispatch table lookahead. Not reactively during the burst. Before it.

The FSM has no transitions that go from EXECUTE back to PLAN mid-window. A circuit that exceeds its declared budget is a compile-time impossibility — if it compiled, the compiler proved it fits. The FSM is deterministic by construction.

**A circuit waiting for data generates no instructions.** In a conventional OS, an application waiting for network data calls `recv()`, blocks on a socket, and is put to sleep by the scheduler — a context switch out, a wait queue entry, a wakeup later, a context switch back in. All of that has cost: saving register state, scheduling overhead, cache eviction, TLB state loss, wakeup latency. The application is present in the system as a sleeping task consuming memory and scheduler state, doing nothing, waiting to be woken.

In the clock-aware model, a circuit whose input channel has no data simply has no window in the dispatch table for that tick. It is not sleeping. It is not spinning. It is not in a wait queue. It generates no instructions to the CPU because it does not appear in the dispatch table at all. No register state to save. No cache eviction. No wakeup mechanism. No scheduler notification. The circuit is absent — not blocked, not sleeping, but structurally not present in that tick's execution. When data arrives into its declared channel, the dispatch table includes its window in the next applicable tick and it executes. The transition from absent to executing costs exactly the ticks between window boundaries — nothing more.

### How Cycles Work: What the Compiler Solves

This is the mechanism. The question "how does the system actually run by cycles" has a precise answer at every level.

**Step 0 — Read the hardware.**

Before translating a single declaration, the compiler needs two numbers from the hardware: clock frequency and pipeline depth. Both are available without guessing.

On x86, `CPUID` leaf `0x15` returns the TSC frequency directly in Hz. On ARM, the system register `CNTFRQ_EL0` holds the counter frequency. On either architecture, the BIOS/UEFI exposes the same figure in the ACPI `FADT` table. The compiler reads it at build time — not at boot, not at runtime. The frequency is a fact about the physical chip, declared in silicon, readable before the first instruction of the programme executes.

Pipeline depth is equally concrete. A modern CPU pipeline is a fixed number of stages: Skylake is 14–19 stages, Cortex-A76 is 13. This is published in the Intel/ARM optimisation manuals and also exposed in `cpuid`'s model-specific data. Pipeline depth tells the compiler two things: the minimum instruction latency for a dependent chain (you cannot issue the next instruction until the pipeline has resolved the previous one's result), and the penalty for a branch misprediction (pipeline flush = N stages × 1 tick). Both affect worst-case instruction count. Both are known statically for a declared `cpu_model`.

The programmer does not read any of this. It goes into `system.cap` once, as `cpu_model = "intel-skylake-3ghz"`, and the compiler's model for that CPU supplies frequency, pipeline depth, execution unit counts, store-forward latency, and cache-line size — the full machine model. If `cpu_model` is not in `system.cap`, the compiler refuses to produce a binary. Timing correctness cannot be proved without knowing the machine.

**Step 1 — Translate declarations to ticks.**

`@Timeslice(core = 2, cycle = "4ns", budget = "3.5ns")` says: on core 2, this circuit has a 4 ns window and must finish within 3.5 ns. The compiler reads `cpu_model` from `system.cap` — say, 3 GHz — and translates: one cycle = 0.33 ns, so 4 ns = 12 ticks, budget = 10.5 ticks. All timing from this point is in integer tick counts, not nanoseconds. The hardware knows nothing about nanoseconds.

This translation is a one-time operation, performed once at compile time, and never repeated. From that point every layer of the system speaks the same unit: **1 `CLKIN` tick = 1 cycle**. The compiler counts instructions in ticks. The declared budget is an integer tick count. The `llvm-mca` instruction cost model returns ticks. The dispatch table stores window boundaries as tick offsets from the system epoch. The hardware counter (`CNTVCT_EL0` / `RDTSC`) increments in ticks. The runtime compares the counter to the dispatch table entry — both in ticks, no conversion, no approximation, no floating-point rounding accumulated across the stack. A method whose worst-case instruction count is 10 ticks costs exactly 10 ticks. The budget that declares it must fit in 12 ticks is exactly 12 ticks. The comparison is integer equality. There is no fuzziness at any layer.

**Step 2 — Count the instruction cycles.**

The compiler feeds the compiled circuit body to `llvm-mca` (or its equivalent in the self-hosted toolchain). `llvm-mca` models the CPU's execution units, pipeline depth, and memory latencies for the declared `cpu_model` and returns the worst-case instruction count. Instruction ordering matters here: the compiler's backend arranges the instruction sequence to avoid pipeline stalls — an instruction whose input depends on the output of the instruction N stages ahead must be separated by at least N ticks, or a stall fills those ticks with nothing. The compiler knows N because it knows the pipeline depth from Step 0. It reorders, fills gaps with independent instructions, and only fails with a compile error if the worst-case count — after optimal ordering — still exceeds the budget.

Two costs are added on top of the raw instruction count before the budget check runs:

**Pipeline fill cost.** At the start of every window, the CPU pipeline is empty — the previous circuit's last instruction has already committed and the pipeline stages are idle. Before the first result of the new circuit is available, the pipeline must be filled: on a 4-stage in-order pipeline (simple ARM cores), that is 4 ticks; on a Cortex-A76 (13 stages) or Skylake (14–19 stages), it is more. The compiler reads the pipeline depth from the `cpu_model` declaration and adds it as a fixed overhead at window start. A circuit whose instruction body costs 10 ticks on a 4-stage pipeline has a true window cost of 10 + 4 = 14 ticks. The budget must cover both. A circuit that declares a budget of 12 ticks on a 4-stage pipeline has only 8 ticks available for its instruction body — the pipeline fill is pre-charged, not hidden.

**Runtime dispatch overhead.** The runtime's own instructions run between windows: read `CNTVCT_EL0` (or `RDTSC`), compare to the dispatch table entry, branch to the circuit's entry point, and at window close read the counter again and write the delta to the observability channel. These instructions are not zero-cost. Their tick count is known statically — the runtime is compiled by the same toolchain against the same `cpu_model`. The compiler measures the runtime's inter-window overhead at build time and reserves it as a mandatory gap between every consecutive window pair in the dispatch table. A circuit whose window closes at tick N does not hand off to the next circuit at tick N — it hands off at tick N + runtime_overhead_ticks, where `runtime_overhead_ticks` is a compile-time constant derived from the runtime's own instruction count. This gap appears in the dispatch table as dead ticks owned by the runtime, not by any circuit. The constraint solver in Step 3 treats these gaps as fixed, non-negotiable reservations.

**Step 2a — Two compiler passes before code emission.**

Before emitting any instruction sequence, the compiler runs two passes over the full circuit graph:

**Pass 1 — Data flow graph: memory order handoff (what flows where).** The compiler walks every declared channel connection across the system — which circuit produces to which channel, which circuit consumes from it, at which declared tick. This produces the complete data flow graph: every value, its source circuit, its destination circuit, its lifetime tier, and the tick ordering constraint between them (`writer_window_end < reader_window_start + coherency_gap`). This pass does not emit instructions. It determines what exists and what must flow where before any code generation decision is made. A channel connection that cannot satisfy its ordering constraint is a compile error at this pass — before instruction counting begins.

**Pass 2 — Code generation: cycle contract, register reuse, speculative pipelines.** With the data flow graph established, the compiler generates instructions in priority order:

1. **Cycle contract (hard floor)** — Prove that the instruction count (including pipeline fill and runtime overhead from Step 2) fits within the declared budget. If it does not after optimal instruction ordering, it is a compile error. Nothing else runs until this passes.
2. **Register reuse (attempt zero-cost delivery first)** — For same-core consecutive windows, keep produced values in the register file across the window boundary. No store, no load, no memory traffic. The compiler attempts this for every handoff before falling back to tier memory. Only cross-core handoffs or values whose lifetime exceeds the register file's capacity fall through.
3. **Speculative pipelines (spend remaining budget)** — After the non-speculative instruction count is known and the cycle contract is satisfied, the compiler checks whether both arms of every hot branch fit within the remaining budget. If yes, both arms are emitted into `SPEC_A` and `SPEC_B` regions. If not, sequential evaluation is used. This pass is last because it depends on knowing the full instruction count from the non-speculative path first — you cannot know how much budget is left to spend on speculation until the contract for the guaranteed path is proved.

**Step 3 — Build the static dispatch table.**

The compiler collects every circuit in the system — OS circuits and application circuits — and solves for a static, non-overlapping assignment of tick windows to cores:

```
core 0, tick   0: ClockCircuit      (window: 6 ticks)
core 0, tick   6: MemoryCircuit     (window: 8 ticks)
core 2, tick   0: parsePrice        (window: 12 ticks)
core 2, tick  12: updateBook        (window: 10 ticks)
core 3, tick   0: ObservabilityCircuit (window: 300M ticks)
```

This is a constraint satisfaction problem: assign windows such that (a) no two circuits share a core at the same tick, (b) every window ≥ its `llvm-mca` worst-case count plus pipeline fill cost, (c) every consecutive window pair has a mandatory gap ≥ `runtime_overhead_ticks`, (d) every handoff pair satisfies writer_window_end < reader_window_start + coherency_gap. If the constraint system is unsatisfiable — the declared circuits do not fit on the declared cores within their declared budgets — it is a compile error, naming the conflicting circuits and the shortfall in ticks. The system does not ship with a timing conflict hidden inside it.

Critically, the OS circuits are in this table. Because the OS is written in the same language, compiled by the same compiler, the compiler knows the timing of `ClockCircuit`, `MemoryCircuit`, `NicCircuit`, and every other OS circuit with the same precision it knows `parsePrice` or `updateBook`. There is no hidden layer. There is no "kernel time" subtracted from user budget as a runtime guess. The OS circuits occupy declared tick windows; the compiler reserves those windows first; application circuits fill the remaining capacity. The constraint solver sees the complete picture: every tick on every core, OS and application together, as one unified scheduling problem.

This is qualitatively different from every existing OS scheduler. Linux's `SCHED_DEADLINE` takes a declared execution time and deadline and attempts to honour them at runtime — but it cannot prove the OS itself will not violate them, because the kernel's own execution time is not declared. The clock-aware compiler proves the OS and the application fit together before either executes.

The dispatch table is emitted as a static data structure in the binary. It is not computed at boot. It is not modified at runtime (unless a new circuit is added dynamically, which recomputes and re-validates the affected core's assignment). The table is a theorem: every entry has been proven non-conflicting at compile time.

**Step 4 — Execute against the hardware clock.**

The CPU has a cycle counter accessible in one instruction: `RDTSC` on x86, `CNTVCT_EL0` on ARM. This is not a software timer. It is a register that increments with every hardware clock tick. The runtime reads it once at startup to anchor the epoch. Every circuit entry point reads the counter, subtracts the epoch, and asserts it is within the declared window. Every circuit exit reads the counter again — the delta is the actual execution time, written to the observability channel.

When a circuit finishes before its window closes, the runtime busy-waits on the counter until the window boundary. Not `sleep`. Not `yield`. Not an interrupt. The CPU spins on a single-instruction counter read until the tick count matches the window boundary. This is not wasteful — the alternative is a context switch, which costs hundreds of nanoseconds and destroys cache state. On a dedicated core, spinning on RDTSC is the correct and intended use of the CPU.

When a new circuit is added dynamically (a program launch), the compiler re-solves the constraint table for the affected cores, validates the new assignment, and hot-patches the dispatch table. The existing circuits continue executing their windows unchanged. The new circuit's windows begin at the next unallocated tick boundary. No reboot. No pause. The clock did not stop.

The runtime adapts the new circuit's declared windows to the live system's current tick offset. The dispatch table is anchored to an epoch — the `RDTSC` value read at boot. Every window is expressed as `epoch + N ticks`. When the new circuit arrives, the runtime reads the current counter, computes the offset into the current epoch cycle, and aligns the new circuit's first window to the next available tick boundary past that offset. The new circuit does not start at tick 0; it starts at the correct tick for a system that has been running. Its declared timing is preserved exactly — only the absolute start tick shifts to fit the live clock domain. Every subsequent window follows at the declared interval from that anchor. The circuit is phase-locked to the running system from its first cycle.

The two phases are distinct and must not be confused:

**Compile time — compatibility is verified.** The compiler receives the incoming circuit manifest and the current system manifest (all circuits already running, their windows, their channel subscriptions). It re-solves the constraint table: does the new circuit fit? Do its windows conflict with any existing window on the target cores? Do its declared handoffs — reads from channels produced by existing circuits — satisfy writer_window_end < reader_window_start given the live offsets? If any constraint is violated, the launch is rejected before a single instruction of the new circuit executes. The system state does not change. The running circuits are not disturbed.

**Runtime — slotting.** If the compiler says compatible, the runtime's job is mechanical: read the live counter, compute the phase offset, insert the new window boundaries into the dispatch table, register channel subscriptions. No arbitration. No negotiation. The compatibility was already proved. The runtime executes a theorem, not a policy.

When multiple applications are already running, their interactions are exclusively through declared memory handoffs — values written at a declared tick boundary by one circuit and read at a declared tick boundary by another. The compiler already proved those handoffs valid for every existing circuit pair. Adding a new circuit that connects to an existing circuit via a declared `channel T` extends the handoff graph by one edge. That edge is verified at compile time against the live timing: the new circuit's read window must open after the existing circuit's write window closes, accounting for the live phase offset. If it does, the handoff is valid by the same theorem as every other handoff in the system. There is no runtime arbitration — the memory location is not contested. One circuit writes at tick N. One circuit reads at tick N+k. The compiler proved k ≥ coherency latency. The hardware clock enforces it. The handoff is the arbitration.

### The Performance Mechanism: Pipelining

Cycles executing in order is not a constraint — it is the source of performance. The dispatch table is a software pipeline. Each circuit is a stage. The clock advances the pipeline. The compiler's job is to arrange the table so that data is always exactly where the next stage needs it, at the tick the next stage starts.

The mechanism depends on the declared lifetime tier of the value being handed off:

**Register-based handoff — same core, consecutive windows.**

If the value's lifetime is `Register` or `ephemeral`, it never needs to leave the CPU's register file. The compiler arranges consecutive windows on the same core so that the output registers of window N are the input registers of window N+1. No store. No load. No memory traffic at all. The handoff costs zero cycles because the data is already in the right place. This is identical to how a hardware pipeline passes a value between stages through a flip-flop: the register is the pipeline register.

**Memory-based handoff — same core, higher lifetime.**

If the value's lifetime is `task` or `session`, it lives in L1 or L2 (pinned). The compiler emits the store at the last instruction of window N. The reader's window N+1 opens on the same core immediately after. The store-to-load forwarding path in the CPU delivers the value from the store buffer before it even reaches L1 — the latency is zero observable cycles for the reader. The compiler knows the store-forward latency for the declared `cpu_model` and schedules the window gap to be ≥ that latency. The latency is absorbed in the gap between windows, charged to neither circuit's budget.

**Cross-core handoff — different cores, same or different dies.**

The compiler reads the cache topology from `system.cap`: same L3 domain, different L3 domain, or NUMA. Each topology has a known coherency latency in ticks. The compiler adds a mandatory gap between the writer's window close and the reader's window open on the other core, equal to the coherency latency for that topology. The reader's window does not open until the data is guaranteed coherent. The handoff latency is a theorem in the dispatch table — not a runtime measurement, not a best-effort estimate, a verified tick count.

**RAM-direct handoff — bypassing cache when the window budget allows.**

For large `permanent` or `Cold`-tier values — reference tables, ML weight tensors, bulk datasets — the cache hierarchy is not the right path. These values do not fit in L1 or L2. Normally this means cache pressure and eviction. In the clock-aware model, the compiler knows the declared size of every value and its tier. If a circuit's window budget is large enough to absorb DRAM latency directly — typically 100–300 ticks for a local DRAM access at declared physical address — the compiler routes the access directly to RAM, bypassing the cache hierarchy entirely. No cache line is brought in. No cache line is evicted to make room. The DRAM access is declared, timed, and absorbed in the window gap. Cache pollution from large cold reads is structurally impossible — the compiler never routes them through L1 in the first place. The cache is reserved for what is proven to be hot.

**The system as a whole is a deeply pipelined machine.**

```
core 2, tick  0–12:  parsePrice     → produces channel Price   (Register lifetime)
core 2, tick 12–22:  updateBook     → consumes channel Price   (same core, register handoff, 0 cycles)
core 2, tick 22–34:  emitQuote      → consumes OrderBook state  (task lifetime, L1 pinned)
core 0, tick 14–30:  riskCheck      → consumes channel Price   (cross-core, gap ≥ coherency latency)
```

The compiler proves this pipeline is valid: every stage receives its input before its window opens, every stage finishes before its window closes, every handoff latency is absorbed in a compiler-computed gap. No pipeline stall is possible — because a stall would mean a circuit exceeded its budget, which is a compile error.

This is what Vivado does with FPGA timing constraints: it solves for register-to-register paths, verifies each path meets the clock period, and reports setup/hold violations at synthesis time. The clock-aware compiler does the same thing in software, for a CPU, with a static dispatch table instead of a place-and-route netlist.

The CPU's own hardware pipeline — out-of-order execution, store forwarding, prefetcher — runs inside each window, invisible to the compiler. The compiler's pipeline runs across windows. Two levels of pipelining, both proven, both counted.

### Register Forwarding and the Multi-Stack Pipeline

The runtime tracks register file state across window boundaries. When a circuit completes its window, the runtime records which registers hold live values — values that the next circuit on the same core declares as inputs. Those registers are not clobbered between windows. The next circuit's first instruction reads directly from the register the previous circuit's last instruction wrote to — no store, no load, no memory traffic. This is **software register forwarding**: the runtime maintains a per-core register liveness map, derived from the manifest's declared channel types, and the compiler emits instruction sequences that assume the forwarded register state is already present. Register-lifetime handoffs cost zero cycles not by coincidence but by construction.

Each circuit executes on its own declared stack — not the hardware stack. The hardware stack pointer (`SP`) is reserved exclusively for the runtime's dispatch loop itself: a fixed, small, runtime-only stack that never grows during normal circuit execution. Every circuit has a declared stack region in its manifest, pre-allocated in the `task` tier (L1/L2 pinned) before the circuit's first window opens. The runtime maintains a **multi-stack pipeline**: each slot in the dispatch table has its own base pointer into its pre-allocated stack region. Switching between circuits requires no register-save/restore — the next circuit's registers are already known from the forwarding map, and its stack is already present in L1. A circuit transition costs exactly the ticks between window boundaries. Nothing more.

### Stack Memory, Speculative Memory, and the Flat Address Map

The multi-stack model has a direct consequence for how physical memory is partitioned. Each circuit slot in the dispatch table owns three distinct physical regions, all declared in the manifest and pre-allocated before the slot's first window:

```
  Per-circuit physical regions (pre-allocated at slot time)

  ┌─────────────────────────────────────────────────────┐
  │  LIVE REGION (task tier, L1/L2 pinned)              │
  │  Stack frame + declared task-tier channel buffers   │
  │  Written by this circuit, read by declared consumers│
  │  Physical base: manifest.live_base                  │
  ├─────────────────────────────────────────────────────┤
  │  SPECULATIVE REGION A (task tier, L1 pinned)        │
  │  Branch arm A — compiler evaluates both sides       │
  │  Written during window, discarded if arm B wins     │
  │  Physical base: manifest.spec_a_base                │
  ├─────────────────────────────────────────────────────┤
  │  SPECULATIVE REGION B (task tier, L1 pinned)        │
  │  Branch arm B — the other side of every hot branch  │
  │  Written during window, discarded if arm A wins     │
  │  Physical base: manifest.spec_b_base                │
  └─────────────────────────────────────────────────────┘
```

**The live region** is the circuit's primary working memory — its declared stack, its `task`-tier channel read buffers, its intermediate `ephemeral` values. This is what a conventional stack holds, but pre-allocated at a known physical address, pinned to L1, and never dynamically grown.

**The speculative regions** are where the compiler's compile-time branch speculation lands. When the compiler evaluates both sides of a branch — because both sides fit within the declared budget and neither produces side effects that cannot be isolated — it emits instructions that write each arm's result into its own dedicated physical region. The two regions are computed in parallel (or in declared instruction-wave order). At the cycle boundary, the runtime reads the branch condition result and commits the winning arm's output to the live channel. The losing arm's region is discarded — it is overwritten at the next window without any reclaim instruction, because it was never part of the circuit's declared live state.

```
  Speculative evaluation during a window

  branch condition ─┬─► ARM A instructions ──► SPEC REGION A
                    │                               │
                    └─► ARM B instructions ──► SPEC REGION B
                                                    │
  at window boundary:                               │
  condition resolved ──► winning arm ─────────────►│──► live channel
                         losing arm   discarded ◄───┘
```

This is not hardware speculation. Hardware speculation occurs at runtime, speculatively, with a misprediction penalty on the wrong guess. This is **compiler-directed static evaluation** of both arms at declared cycle positions, with a deterministic commit at the cycle boundary. The two speculative regions are physical addresses the compiler chose at build time. The commit is a manifest-declared write to a live channel address. No branch predictor is involved. No pipeline flush on a wrong guess. The cost of evaluating both arms is paid once, deterministically, within the declared budget — and the compiler proved it fits before emitting the code.

The speculative region size is part of the circuit's manifest footprint. The compiler calculates the maximum output size of each arm — from the declared types of the channels those arms produce — and reserves exactly that much space in each speculative region. The runtime includes both speculative regions in the circuit's pre-allocation. They count against the circuit's declared `task`-tier footprint and are verified against L1 capacity at compile time, the same as every other declared value. Speculative evaluation has no hidden memory cost. It is declared, bounded, and compiler-verified like everything else.

The physical address map for the whole system is therefore the sum of every circuit's three regions, plus their `session` and `permanent` tier allocations, plus the runtime's own tables. The entire physical address space is known before the first instruction executes. The runtime does not discover it. It reads the compiled address map, issues the pre-allocations, wires the DMA descriptors, and begins the dispatch loop. The memory layout is a compile-time theorem. Execution is the proof that the theorem holds.

### The Trace Unit — Proving the Compiler Was Right

On ARM (and equivalent architectures), the trace unit timestamps every instruction as it commits. The clock-aware runtime subscribes to the trace unit's output as a `channel TraceEvent` through the `ObservabilityCircuit`. This gives the system something no conventional OS has ever had: a continuous, hardware-sourced, instruction-level audit of every circuit's actual execution.

What the trace unit provides:

| Observable | What it proves |
|---|---|
| Instruction commit timestamp | Actual cycle at which each instruction executed |
| Register values at commit | Register file state at any point in time |
| Branch taken / not taken | Confirms compiler's branch elimination was correct |
| Cache miss events | Proves zero cache misses on the hot path (or flags a violation) |
| Port utilisation per window | Confirms actual execution unit usage matches compiler model |
| Speculation events | Confirms no speculative execution occurred outside declared bounds |
| Pipeline stall events | Confirms no circuit exceeded its declared budget |

The compiler emits an **execution plan** alongside the binary — the expected sequence of instruction timestamps, register values, and port utilisations for the hot path. The `ObservabilityCircuit` continuously compares the trace stream against the execution plan. A deviation is a `channel ExecutionPlanViolation` signal: the specific instruction, the expected cycle, the actual cycle, and the delta.

This produces three guarantees that no conventional system can make:

1. **Zero cache misses, proven in real time.** Every cache miss on the hot path is a trace event. If the execution plan predicts zero misses and the trace shows one, a violation signal fires immediately — not in a post-mortem profiler, not in a benchmark report, in the same execution window.

2. **Misprediction confirmation.** There are no branches on the hot path by construction. If the trace unit reports a branch prediction event in a circuit whose hot path the compiler eliminated all branches from, it confirms the compiler model is correct — or flags a hardware anomaly in the CPU itself.

3. **Execution plan validation.** A circuit that consistently executes faster than its declared budget can donate tick windows back to the runtime pool. A circuit that consistently approaches its budget boundary signals the `ObservabilityCircuit` before it becomes a violation. The runtime reacts through declared channels, not through forced intervention.

On ARM specifically, the ETM (Embedded Trace Macrocell) provides this at silicon level with sub-cycle resolution. The clock-aware runtime treats it as a first-class subscription source — not a debug facility to be attached occasionally, but a permanently running circuit producing a continuous stream of hardware-sourced proofs.

The trace unit is also the input to PGO. Real execution traces from production replace synthetic profiles. The next compilation ingests the trace, adjusts instruction ordering and register allocation to match actual hotspots, and produces a binary that is provably better than the previous one — not heuristically better. The trace is a theorem input.

### `@Measure` — Zero-Scaffolding Benchmarking

Every circuit window already has a start tick and an end tick — the hardware counter values at window open and window close, read by the runtime as part of its normal dispatch-loop evaluation and recorded in the atom stream. `@Measure` is the annotation that names one of those measurements and declares that the `ObservabilityCircuit` should accumulate it into a rolling percentile distribution, exposed as a typed channel.

```
@Measure("parsePrice")
circuit ParsePrice {
    @Timeslice(core = 2, cycle = "4ns", budget = "3.5ns")
    fn parse(frame: EthernetFrame) = ...
}
```

That is the entire benchmark declaration. No benchmark harness. No warmup loop. No separate profiling binary. No external tooling. No library to link. No import statement. The annotation tells the compiler two things:

1. **Name this measurement** — the string `"parsePrice"` becomes the key in the `ObservabilityCircuit`'s measurement registry.
2. **Generate a percentile channel** — the compiler produces `channel ParsePriceMeasurement` automatically, backed in the `session` tier. The `ObservabilityCircuit` writes to it at its declared low-frequency window.

The generated channel holds the full raw sample set — every `CLKIN` tick delta from every execution, in the order they occurred:

```
channel ParsePriceMeasurement {
    val element = Tick          // one CLKIN tick delta per execution, as recorded
    val tier    = permanent     // accumulated for the programme lifetime
    val size    = 16_777_216    // declared capacity — 16M samples at 8 bytes = 128 MB
}
```

Reading is reading:

```
val samples = ParsePriceMeasurement.get()
```

Computing a percentile is calling a function on the samples:

```
val p99   = percentile(samples, 99.0)
val p9999 = percentile(samples, 99.99)
val p50   = percentile(samples, 50.0)
val tail  = percentile(samples, 99.999)   // any quantile, full fidelity
```

Pre-bucketing into p50/p90/p99 at the OS level would bake in a decision about which quantiles matter — and that decision is always wrong for someone. A pre-bucketed histogram with 1000 bins cannot compute p99.9 accurately if the tail is sparse; it cannot compute a novel quantile at all after the fact. Raw samples have none of these limitations. Every quantile is computable. Full fidelity. No precision loss at the tail.

The `size` declaration is the programmer's statement of how many samples to retain. The `ObservabilityCircuit` writes each tick delta into the channel as a plain `Tick` value at the circuit's window close — one write per execution, in the channel's declared `permanent` tier, which is never paged out. When the channel reaches capacity the oldest samples are overwritten in ring-buffer order — the most recent N executions are always available. The size is declared in the manifest and verified against physical DRAM capacity at compile time, the same as every other `permanent` value.

You run the code — in production, on real data, under real load — and you read the channel. There is no synthetic harness, no warmup phase, no separate measurement binary. The measurement is the programme.

**Why this is categorically different from conventional benchmarking.**

A conventional benchmark runs synthetic workloads in isolation under controlled conditions, with a warmup phase to prime the cache. It measures a proxy for production behaviour. The proxy is imprecise because the cache state at benchmark start differs from production, the data is synthetic rather than real, and the benchmark harness's own instruction mix appears in the measurement window. The result is a number that approximates production timing but cannot be the same thing.

`@Measure` measures production execution. The circuit is already running in its declared window, against real inputs, with its working set already pinned to L1 by the runtime before its first instruction. The start and end `CLKIN` ticks are read by the runtime as part of normal dispatch-loop EVALUATE — the measurement adds zero overhead to the circuit's own budget. The `ObservabilityCircuit` writes the delta in its own declared window, isolated from the hot path. The sample set is every actual production execution, not a synthetic proxy.

The measurement infrastructure is not a library. It is the OS. The `ObservabilityCircuit` is already running — it was declared in `system.cap` at boot alongside `ClockCircuit` and `MemoryCircuit`. The trace unit is already streaming `TraceEvent` atoms. The tick counters are already being read every window. `@Measure` does not introduce new machinery — it names a measurement point within infrastructure that already exists and was already running before your circuit was slotted. You do not pay for it. You do not import it. You do not configure it. You annotate and the OS does the rest, precisely, in hardware, at `CLKIN` resolution, from the first execution to the last.

**Intra-window checkpoints.** If the full-window sample is insufficient and you need to locate where within a window the time is spent:

```
fn parse(frame: EthernetFrame) = {
    val header  = parseHeader(frame)
    @Checkpoint("headerParsed")
    val decoded = decodePayload(header)
    @Checkpoint("payloadDecoded")
    val price   = extractPrice(decoded)
    price
}
```

Each `@Checkpoint` annotation inserts a single `CNTVCT_EL0` (or `RDTSC`) read at that label in the instruction sequence. The compiler includes the cost of that read — one instruction — in the budget analysis. The checkpoint emits a delta to `channel ParsePriceCheckpoints`, one `Tick` per label per execution. Each label gets its own sample set, accumulated in the same `permanent` ring buffer. You call `percentile` on any label's samples the same way:

```
val headerSamples  = ParsePriceCheckpoints.get("headerParsed")
val payloadSamples = ParsePriceCheckpoints.get("payloadDecoded")
val p99Header      = percentile(headerSamples, 99.0)
```

**Measurement channels are first-class subscribers.** Any declared circuit can subscribe to a measurement channel. An alert circuit reads the raw samples and applies whatever condition it declares:

```
circuit ParsePriceAlert {
    fn evaluate(samples: ParsePriceMeasurement) = {
        val tail = percentile(samples, 99.9)
        match tail > budget * 0.9 {
            true  => AlertSignal.put(ParsePriceBudgetWarning { tail, budget })
            false => ()
        }
    }
}
```

The alert circuit runs in its own declared window, reads the sample channel, computes the quantile it cares about, and emits through the normal channel mechanism. It is not constrained to fixed percentiles declared at annotation time — it computes whatever the circuit logic requires, from the full sample set, at the precision the full sample set allows. No polling. No external monitoring system. No Prometheus scrape interval. The samples are in the system. The analysis is in the system. The channel is the interface.

---

