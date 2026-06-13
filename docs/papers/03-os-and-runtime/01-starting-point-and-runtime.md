## The Starting Point

Paper II defines a language with four rules: clock annotation, lifetime type, channel, exhaustive match. If it compiles, it is hardware-correct. The question this paper answers is: what happens to the OS, the runtime, and the kernel when everything above the hardware is written in that language?

The answer is that the distinctions dissolve. The runtime is the kernel is the scheduler is the memory manager. They are all collections of declared-timing circuits, compiled by the same toolchain, verified by the same compiler, slotted into the same dispatch table by the same runtime. There is no separate layer. There is no separate concern. There is one model, applied consistently from the boot stub to the application.

---

## The Runtime Is a Bitstream Loader

In a conventional stack, the runtime is the layer between the language and the OS: it manages the heap, schedules goroutines or threads, runs the GC, and calls into the kernel via syscall. The runtime and the OS are distinct because they were written by different people for different purposes.

In a clock-aware stack, the distinction dissolves.

The closest analogy in hardware is a bitstream loader. An FPGA bitstream loader receives a compiled netlist — routing, LUT configuration, clock constraints, I/O assignments — and programs it onto the fabric. It does not interpret the netlist at runtime. It does not arbitrate between competing netlists. It loads the configuration, the fabric executes it, and the loader steps aside. The runtime in the clock-aware model is this exactly: it receives a compiled circuit manifest — timing windows, lifetime declarations, channel subscriptions, cryptographic identity — programs it into the dispatch table, pre-allocates the declared memory, and steps aside. The circuit executes. The runtime does not intervene again unless a signal arrives on a declared channel.

The compiler is Vivado. The manifest is the bitstream. The dispatch table is the fabric. The clock is the clock. The analogy is not approximate — it is the same model applied in software.

**The FPGA analogy exposes the fundamental difference between the two substrates — and why the window concept is the correct bridge.** In an FPGA, parallelism is *spatial*: more LUTs means more logic executing within a single clock cycle. A pipelined FFT on an FPGA stages each butterfly operation into a separate slice of LUTs so that every clock edge advances all stages simultaneously — 64 operations per cycle, because 64 sets of LUTs are wired in parallel. The clock cycle is fixed; the LUT count is the degree of freedom. A CPU has no equivalent freedom. It has a fixed, small number of execution ports — integer ALUs, FP units, load/store units — and that number is a silicon constant, not a programmer variable. You cannot instantiate more execution ports in software.

The window is the CPU's answer. Where an FPGA trades silicon area for work-per-cycle, a clock-aware CPU trades declared time — `budget_ticks` — for work-per-period-boundary. The programmer does not write `use 8 LUTs in parallel`; they write `@Timeslice(period = "1ms")`. The compiler then proves that the circuit's entire instruction graph, sequenced through the CPU's fixed set of execution ports, completes within `budget_ticks` cycles. The window is not a timeout. It is the exact temporal footprint of the circuit on the CPU, proved by the compiler, guaranteed to be non-overlapping with every other circuit's window. It is the CPU equivalent of routing LUTs to fabric: a statically computed, exclusively allocated region of the execution medium.

**The model is application-level pipelining.** A CPU pipeline stages instructions so that fetch, decode, execute, and writeback overlap — four instructions in flight simultaneously, each in a different stage, each completing one stage per cycle. The clock-aware dispatch table does the same thing one level up: circuits are the stages, windows are the stages' time slots, and the epoch is the pipeline cycle. `RiskCheck` runs in its window while `PricePublish` — whose input is `RiskCheck`'s output channel — waits at the next stage. `MarketDataIngestion` runs in a parallel stage on a separate core. Multiple applications execute simultaneously, each within its proved window, none overlapping, none blocking. The dispatch table is the pipeline register file: it records which stage executes when, and the compiler has already proved that every stage fits within its slot.

The constraint is identical to CPU pipelining: each stage must complete within its allotted cycles, or the pipeline stalls. In a CPU, a stage that takes too long is a structural hazard — the microarchitect redesigns the stage. In the clock-aware model, a circuit whose instruction graph exceeds its window is a structural hazard at compile time — the programmer splits the circuit, widens the period, or restructures the call graph. The discipline is the same. The level is different. The CPU pipelines instructions; the runtime pipelines circuits. Both produce a system where every stage is bounded, every slot is filled, and forward progress is guaranteed by construction.

**The compiler and the runtime are intentionally separate.** This is not a gap in the model — it is the model's correct factoring. Developers compile on their own machines. The manifest — a signed, self-contained proof artefact — is what ships to the production server. The runtime on the production server never runs the compiler. It receives the manifest, verifies the cryptographic signature against the key ring in its `system.cap`, checks that the declared resource profile fits the production machine's declared capacity, and loads the circuit into the dispatch table. If any check fails, the manifest is rejected before a single window is allocated. The compiler is not present at runtime. It does not need to be. The proof travels with the manifest.

This separation is the same separation that makes hardware distribution work: Vivado runs on the engineer's workstation; the bitstream ships to the FPGA on the target board; the board does not need Xilinx software to run. The compiler's job ends when the proof is signed. The runtime's job begins when the proof arrives. They share nothing except the manifest format and the `system.cap` schema — and both of those are versioned, declared, and verifiable.

The roles are precisely distinct. The compiler **proves feasibility**: given this hardware model, this circuit fits, this budget holds, this L1 footprint is resident, this compliance target is met. The proof is static, complete, and produced once. The runtime **enforces it**: at every clock edge, on every core, in every tick, the dispatch table is executed and the atom stream records what actually happened. The compiler's proof is a certificate of what should happen. The runtime's atom stream is a certificate of what did happen. Both are cryptographically signed. Neither is optional.

The atom stream is therefore a continuous audit log — not a debug facility, not an optional trace, but a permanent, hardware-sourced record of every enforcement decision the runtime made. It is available at all times: to the `ObservabilityCircuit` for live monitoring, to the `AdaptationCircuit` for ML-driven optimisation, to external audit tools that subscribe to `channel AtomStream`, and to post-incident forensic analysis that can replay any tick from any point in the system's history. A regulator, a certifier, or a security auditor can verify, from the atom stream alone, that every declared timing constraint was honoured, every circuit ran under its correct key, and every handoff completed within its declared staleness bound — not as a claim, but as a cryptographically verifiable sequence of hardware-sourced measurements.

The same mechanism is directly valuable to semiconductor manufacturers for yield control. Every chip that runs a clock-aware system is continuously comparing its actual execution latencies against the declared microarchitecture model in `system.cap`. A chip that consistently executes within the declared latency bounds is verified silicon — it is meeting its speed grade in production, under real workload, not just under synthetic test vectors at the factory. A chip that shows a systematic delta — every L1 access costs 1.1 cycles instead of 1.0, every branch unit resolves in 2 cycles instead of 1 — is a chip whose actual silicon characteristics differ from its declared model. That delta is measurable, attributable, and continuous.

The consequence for manufacturers is a feedback loop that has never before existed: field measurements of actual silicon behaviour, at instruction granularity, from every deployed chip, aggregated back to the process characterisation team. Not sampled. Not estimated from test chips. From every production chip, in every deployment, every tick. A systematic shift in field measurements across a production batch identifies a process deviation before it becomes a customer-visible failure. A marginal chip that passes factory test but drifts under sustained thermal load is identified in the field, from the atom stream, before it causes an incident. The atom stream is a continuous, hardware-sourced yield monitor — and it requires no additional silicon, no test equipment, and no factory involvement. It is a consequence of running a clock-aware system.

### The Compiler Produces a Complete Static Resource Profile

When the compiler compiles a circuit it does not only produce executable code. It produces a complete, static, ahead-of-time picture of everything the circuit will ever use during its lifetime. This profile is part of the manifest — the bitstream the runtime loads.

**Port utilisation timeline.** For every channel the circuit subscribes to, the compiler knows exactly when it reads and when it writes: which tick within which window, at what frequency, producing or consuming what type. The profile is a timeline, not a set of bounds (the 847-tick window is the `budget_ticks` proved by the execution window profile below):

```
channel  RawMessage   read    tick   0   of every 847-tick window  (consumer)
channel  Price        write   tick 843   of every 847-tick window  (producer)
channel  HandoffLatency write tick 846   of every 847-tick window  (observability)
```

The runtime loads this profile and hands it to the observability sub-circuit, which uses it as the baseline: if a read or write does not appear at the declared tick, the deviation is itself an observable event. The port utilisation data the observability sub-circuit reports is not sampled — it is the delta between the declared timeline and the measured one.

**Execution window profile.** For every circuit and every function it calls, the compiler walks the full instruction graph, counts CLKIN cycles, adds pipeline drain depth, and emits an exact `budget_ticks` value. This is the window the runtime allocates in the dispatch table:

```
window   circuit  RiskCheck        period = 1ms    budget_ticks = 847   pipeline_depth = 4   core = 1
window   fn       validate         budget_ticks = 312   pipeline_depth = 3   (called from RiskCheck)
window   fn       computeRisk      budget_ticks = 489   pipeline_depth = 4   (called from RiskCheck)
window   fn       writeResult      budget_ticks =  46   pipeline_depth = 1   (called from RiskCheck)
```

The circuit's `budget_ticks` is the sum of every function's cycle count plus the deepest pipeline drain across the call chain, proved by the compiler to be non-overlapping. If the sum exceeds the window implied by `period` — that is, if the function body takes more cycles than the declared recurrence period permits — it is a compile error. The programmer either shortens the function, widens the period, or splits the work across multiple circuits. No runtime measurement resolves this; the proof is either in the manifest or it does not compile.

Every function called within a circuit is inlined or its call overhead is accounted for in the budget. There are no dynamic dispatch costs. There are no virtual method tables, no closure allocations, no GC pauses mid-window. The compiler sees a static call graph and counts cycles across it. The manifest records the result. The runtime allocates exactly `budget_ticks` cycles for the circuit's window — no more, no less.

**Memory footprint by tier.** For every value the circuit declares, the compiler knows its type, its tier, its size in bytes, and its scope:

```
tier     Register    size  8B     scope  expression   count  12 per window
tier     ephemeral   size  64B    scope  function     count  3 per window
tier     task        size  128B   scope  window       count  1
tier     session     size  10MB   scope  circuit      count  1   (RiskLimitTable)
```

The runtime reads this table when slotting the circuit and pre-allocates exactly these amounts from the correct tiers before the first window opens. There is no runtime heap introspection. There is no "how much memory is this process using?" query — the answer was known at compile time and is in the manifest. The runtime's memory accounting is a subtraction: total tier capacity minus the sum of all slotted circuit manifests. Out-of-memory is when that subtraction goes negative, detected at slot time, before any allocation is attempted.

**Physical placement.** The compiler knows — from `system.cap` and the handoff graph — which core each window is assigned to, which cache domain each tier-resident value sits in, and which coherency gap separates each cross-core handoff. This placement is fixed in the manifest. The runtime does not re-derive it. It reads the placement, pins the memory, and programs the dispatch table entry.

The consequence is that the system's complete resource utilisation — across all slotted circuits — is the sum of their manifests. The runtime can answer, at any point, without any measurement: how many ticks on core 2 are allocated; how many bytes of DRAM are committed to `session` tier; which channels are at peak utilisation; which circuits share a cache domain. These are not runtime metrics gathered by a monitoring daemon. They are compile-time theorems, readable from the manifest table. The observability sub-circuit reports deviations from those theorems. The theorems themselves never change while the system is running.

**The runtime derives the total dispatch schedule from the manifest table — before the first window opens.** Because every circuit declares a `period` and the compiler proves its exact `budget_ticks`, the runtime can compute the full timeline for every core by simple arithmetic: for each core, sum the `budget_ticks` of every circuit assigned to it whose period boundary falls within the next epoch. If that sum exceeds the epoch duration in cycles, the system is overcommitted — the new circuit is rejected at slot time, before it disturbs anything running. If the sum fits, the runtime writes the dispatch table: an ordered list of `(circuit_id, start_tick, end_tick)` entries per core, covering the entire epoch. No decisions are made tick-by-tick. The schedule is a static artefact, derived once per epoch boundary from the manifest sums, executed mechanically by the dispatch loop.

This is why adding a circuit is a capacity check, not a scheduling problem. The runtime does not ask "where does this fit?" — it asks "does the sum of budgets still fit within the epoch?" If yes, the new circuit's window is appended to the appropriate core's dispatch list at the position dictated by the channel graph. If no, the circuit is not loaded. The entire complexity of traditional scheduling — priority queues, preemption, time-slicing, fairness policies, starvation prevention — is replaced by one unsigned integer comparison per core per epoch: `Σ budget_ticks ≤ epoch_cycles`. That is the admission test. It either passes or it does not.

### Dynamic Memory Is a Channel with a Declared Size

The obvious objection: real applications have dynamic memory. User input is variable length. Network payloads vary. A JSON document arriving over a socket has no fixed size. A compile-time resource profile cannot accommodate data whose size is not known until runtime.

The resolution is already in the model: dynamic data flows through a `channel T`. A channel has a declared size — the ring buffer capacity — fixed at compile time. The data inside the channel is bounded by that size. If the payload is larger than one channel element, it is segmented — multiple elements, same channel, declared capacity. The channel is the declared bound; the content is the runtime value within that bound.

This is not a restriction. It is a clarification of what "dynamic" means. In a conventional system, dynamic memory means: the programmer does not know at compile time how much memory will be needed, so the runtime must provide it on demand from a heap of unknown size. In the clock-aware model, "dynamic" means: the value's content is not known at compile time, but its maximum size is declared — because a circuit that could consume unbounded memory has an undeclared timing contract, and an undeclared timing contract is a compile error.

The practical consequence:

| Conventional dynamic pattern | Clock-aware equivalent |
|---|---|
| `Vec<u8>` growing unboundedly | `channel Frame` with declared `size = 4096` |
| `String` of unknown length | `channel Byte` or segmented `channel Chunk` with declared capacity |
| `HashMap` growing at runtime | `FlatMap<K,V>` with declared `size` — maximum key count declared |
| Dynamic JSON parse tree | `channel JsonToken` — streaming tokeniser, fixed buffer, no tree allocation |
| Recursive data structure | Iterative traversal over declared-size collection — recursion depth is timing; declare it |

A streaming JSON parser is the clearest example. A conventional parser builds a tree on the heap — unbounded allocation, unknown depth, unknown lifetime. A clock-aware parser reads tokens from `channel RawByte` and writes structured events to `channel JsonToken` — one token at a time, fixed buffer sizes, no allocation, no tree. The application receives a stream of typed events and processes each one in its declared window. The document may be arbitrarily large; the memory footprint is constant and declared.

For genuinely unbounded data — a file upload with no declared maximum, a stream that never terminates — the channel's declared `size` is the processing window. The circuit processes `size` elements per window and requests more in the next window. The throughput is bounded by the declared window frequency, not by memory. The circuit's memory footprint remains constant regardless of total data volume.

The model does not eliminate dynamic data. It eliminates dynamic memory — the condition where memory size is unknown at compile time. Every circuit processes within a declared footprint. The data flowing through it may be arbitrarily variable; the container it flows through is not.

### Where the Model Requires Honest Discipline: SQL and Full Materialisation

SQL is the hardest case. A query `SELECT * FROM orders WHERE date > '2024-01-01'` may return 10 rows or 10 million. The result set size is not known at compile time and cannot be declared without knowing the data. This is a genuine tension.

The model resolves it cleanly for the majority of SQL operations — and honestly does not for one specific case.

**What works without compromise:**

Streaming queries. The database circuit produces `channel Row` with a declared buffer size — say, 1024 rows. The consumer processes each batch in its declared window and reads the next batch in the following window. The total result set may be arbitrarily large; the memory footprint is `1024 × sizeof(Row)`, declared at compile time. The frontend streams the same channel — rendering rows as they arrive, never holding the full result in memory. Aggregations — `COUNT`, `SUM`, `AVG`, `MIN`, `MAX` — run in a single streaming pass. No materialisation. Declared footprint.

**What requires a declared maximum:**

`ORDER BY` on an unbounded result set requires materialising the full set before sorting — unless the data is pre-sorted at write time, or unless the sort is pushed into the database circuit and applied at ingestion. If the application must sort at query time on an arbitrarily large result, it must declare a maximum: `FlatMap<RowKey, Row>` with `size = 100_000`. If the result exceeds that maximum, the circuit emits a `channel TruncationEvent` — a declared signal, handled by the exhaustive match, not a silent overflow. The programmer must decide: declare a maximum and acknowledge truncation, or re-architect the query to push the sort upstream.

This is not a failure of the model. It is the model being honest about what the programme is doing. A sort on an unbounded result set has undeclared memory — the model requires it to be declared. The conventional alternative is a heap that silently grows until the process is killed by the OOM killer. The clock-aware alternative is a declared maximum and an explicit truncation signal. The constraint is real; so is the conventional alternative's behaviour.

**The architectural push:**

The model pushes SQL-style applications toward streaming-first design: declare the maximum result set at the query boundary, push aggregation and ordering into the storage layer where the data is already sorted by write pattern, render the frontend incrementally. This is not a new pattern — it is what high-throughput data systems already do. The clock-aware model makes it the only pattern, rather than the recommended one.

### The Weak Point Dissolves: Every Production System Already Declares Bounds

The apparent weakness of the declared-footprint model — that `session` and `permanent` tier sizes must be declared at compile time — dissolves on inspection. No production system runs with truly unbounded memory. Every deployed JVM has `-Xmx`. Every production PostgreSQL has `shared_buffers`. Every Kubernetes pod has a memory limit. Every embedded system has a linker script with fixed section sizes. The bounds are always declared somewhere — in a config file, a deployment manifest, a command-line flag, an operations runbook.

The difference is where the declaration lives and when it is verified. In a Java application, the heap bound is a runtime flag passed to the JVM at launch — not verified against the programme's actual memory behaviour, not checked against the data structures the programme allocates, not proved consistent with the programme's timing. The flag is an operational constraint applied outside the language. The programme can be written to require more than the flag permits, and nobody will discover this until the JVM throws `OutOfMemoryError` in production.

In the clock-aware model, the bound is declared in the programme itself — as a `session` or `permanent` type with a declared size — and the compiler verifies it is consistent with everything the circuit accesses. The declaration is the same fact the operations team would have put in `-Xmx`. It is just in the right place: in the source, verified at compile time, part of the manifest the runtime loads.

Re-slotting handles growth: when the declared bound proves insufficient at runtime, the circuit signals the runtime, which removes and re-slots it with an updated manifest and a larger declared size. This is the same operation as a rolling restart with a larger `-Xmx` — but initiated by the circuit itself through a declared channel, rather than by an operator running a shell command. The pause is equivalent; the mechanism is structural.

The model does not impose a new constraint. It makes an existing universal constraint visible and compiler-verified rather than hidden in deployment configuration and discovered at runtime.

The memory constraint has always existed. In C it is hidden across hundreds of `malloc` call sites, scattered through library dependencies, invisible until `valgrind` or a production crash surfaces it. In C++ it is hidden behind allocator templates, `std::vector` growth factors, and smart pointer reference counts that individually look innocuous. In Java it is hidden inside the GC's internal bookkeeping, the JIT's code cache, and a dozen framework layers that each allocate on behalf of the application — then surfaced at deployment as a `-Xmx` flag chosen by guesswork and tuned by trial. In every case the constraint exists; the programmer just cannot see it from the source.

The clock-aware compiler derives the constraint from the source — from the declared types, declared sizes, and declared lifetimes — and makes it explicit in the manifest before a single instruction executes. The runtime then acts on that manifest directly: pre-allocating exactly the declared footprint, tracking exactly which circuit owns which range, reclaiming exactly the right bytes at exactly the right time. No guesswork. No tuning. No `-Xmx`. The constraint the programmer always implicitly accepted is now something the compiler computed and the runtime enforces.

### Runtime Arbitrage: Declared Weights, Structural Decisions

Because the runtime holds the complete resource profile of every slotted circuit — tick windows, memory footprint by tier, channel utilisation, priority — it can arbitrage between them continuously and deterministically, without any runtime measurement or heuristic.

The architect declares weights in `system.cap`:

```
circuits.weights = {
    TradingEngine:        100,
    RiskChecker:          90,
    MarketDataIngester:   80,
    ObservabilityCircuit: 10,
    DebugLogger:          5,
}
```

The runtime uses these weights in three situations:

**Resource contention — memory.** When a new circuit cannot be slotted because a tier is full, the runtime evicts the lowest-weight non-exempt circuit first, regardless of how recently it was added or how much memory it holds. The eviction order is fully determined by the weight table. No scoring. No heuristic. The architect declared the priority; the runtime executes it.

**Resource contention — core time.** When a new circuit's declared window cannot fit on any core without displacing an existing circuit's window, the runtime attempts to pack the new circuit alongside lower-weight circuits by compressing their inter-window gaps. If the gap is already at the minimum allowed by the coherency constraints, the new circuit is rejected — or the lowest-weight circuit is evicted to free the core time. The decision is deterministic from the weight table and the declared timing constraints.

**Ongoing arbitrage — adaptive shedding.** The observability sub-circuit continuously compares actual port utilisation against the declared profile. When a circuit consistently underutilises its declared channel bandwidth or finishes well within its budget, the runtime can signal it via `channel ResourceAdvisory` — a declared channel the circuit may subscribe to — suggesting it yield surplus tick windows back to the pool. The circuit's exhaustive match handles the advisory: it may reduce its declared footprint, defer non-critical work to `@Cold` windows, or ignore the advisory if the architect declared that circuit weight-exempt. The decision is always in the circuit's match, not in the runtime's policy.

The architect does not tune a runtime. They declare weights once in `system.cap` and the runtime's behaviour under pressure is fully determined by those declarations. Two deployments with different weight tables produce provably different eviction orders — both correct for their respective declarations. The behaviour under contention is not emergent. It is designed.

This is the system's deepest production property: **deterministic behaviour under any load**. Not "usually deterministic". Not "deterministic unless contention exceeds a threshold". Always. The runtime has no hidden state, no adaptive heuristics, no feedback loops that change behaviour based on history. It has a manifest table and a weight table. Given identical inputs — the same set of slotted circuits, the same incoming circuit request, the same resource state — it makes the same decision every time, on every machine, in every environment. A production incident is reproducible from the manifest snapshot alone. A performance regression is traceable to a specific manifest change. A capacity planning decision produces a provable outcome, not a probabilistic one.

Every other system achieves this only under quiescent load. Clock-aware systems achieve it by construction.

### Multi-Module Projects: Whole-System Verification

The compiler's proof guarantees are strongest when it can see the complete system. A single-module build proves timing correctness for one circuit in isolation. A multi-module build — where every circuit in the deployed system is compiled together against the same `system.cap` — proves timing correctness for the entire system as a composition.

This is the natural project structure the model encourages. Each application is a module: its declared circuits, its channel subscriptions, its `@Handoff` annotations. The system build collects all modules, merges their manifests, and runs the constraint solver once across the full graph:

```
system.modules = [
    OsCircuits,
    MarketDataIngester,
    TradingEngine,
    RiskChecker,
    ObservabilityCircuit,
    DebugLogger,
]
```

What the compiler gains from seeing all modules together:

| Property | Single module | Multi-module whole-system |
|---|---|---|
| Timing proof | Circuit fits its own declared windows | All circuits fit together, no overlap on any core |
| Handoff proof | Channel types match at the declared interface | Writer window provably precedes reader window across module boundaries |
| Memory proof | Circuit footprint fits declared tier | Total footprint of all circuits fits physical memory in `system.cap` |
| Channel permission | Subscription declared in this module | Subscription chain verified end-to-end across all modules |
| Cryptographic coherence | Manifest signed by org key | All cross-module handoffs verified against shared key ring |
| Weight arbitrage | N/A | Eviction order proved correct for declared weight table across all circuits |

A cross-module handoff that the single-module build could not verify — because the producer was in a different module — becomes a whole-system compile error if the timing is wrong. The module boundary disappears at compile time. The circuits compose as if they were written together, because the compiler treats them as if they were.

This also means the system build is the deployable artefact. There is no runtime integration step where modules are composed and hope they work together. The compilation is the integration test. If the whole-system build passes, every cross-module channel is typed, every cross-module handoff is timed, every cross-module memory commitment fits the declared hardware. The system is coherent by construction before it is deployed.

An alternative exists: a project could manually supply external declaration tables — pre-computed timing windows, memory footprints, and channel contracts for each circuit — and the compiler would accept them as axioms rather than deriving them from source. This is not the recommended path. A manually maintained declaration table is a second source of truth. The moment a circuit changes — a new channel added, a lifetime type widened, a budget relaxed — the external table becomes stale. A stale table means the compiler proves a system that no longer exists. The whole-system correctness guarantee is only as strong as the weakest link in the declaration chain. If any declaration is hand-written rather than compiler-derived, the guarantee becomes a convention.

The multi-module build keeps the single source of truth: source code. Every declaration the compiler needs is in the source. Every proof the runtime relies on is derived from that source. The compiler table is not maintained — it is recomputed on every build from the declarations that are already there.

External dependencies are the one legitimate exception. When a circuit depends on a published package — a third-party market data library, a shared risk model — the package ships a pre-compiled declaration table alongside its compiled artefact. This table is not hand-written: it was compiler-derived from the package's own source at publish time, signed with the package author's key, and sealed into the package manifest. The consuming compiler treats it as a trusted axiom — in the same way a linker treats a pre-compiled `.a` as a trusted object — and uses it to verify the cross-module handoffs at the package boundary.

The trust chain is explicit: the package's declaration table is verified against the package's cryptographic signature. A table that does not match the signed manifest is rejected. A package whose declared timing, memory footprint, or channel contracts conflict with the consuming system's `system.cap` is a compile error — not a runtime discovery. The external dependency is integrated with the same proof rigour as an internal module; it just contributes its compiler table rather than its source.

---

