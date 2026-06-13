# Paper III: The OS and Runtime
#### Alex Chevrier — chevrier.alex@gmail.com

> When every operation's timing is declared, the operating system ceases to be a layer and becomes a participant.

This paper follows from [Paper II: The Language](02-language.md). The language defined there is the language the OS is written in. Once the language exists, the runtime, kernel, scheduler, memory manager, driver model, permission system, and deployment model all collapse into the same four rules. This paper traces that collapse.

---

## The Starting Point

Paper II defines a language with four rules: clock annotation, lifetime type, channel, exhaustive match. If it compiles, it is hardware-correct. The question this paper answers is: what happens to the OS, the runtime, and the kernel when everything above the hardware is written in that language?

The answer is that the distinctions dissolve. The runtime is the kernel is the scheduler is the memory manager. They are all collections of declared-timing circuits, compiled by the same toolchain, verified by the same compiler, slotted into the same dispatch table by the same runtime. There is no separate layer. There is no separate concern. There is one model, applied consistently from the boot stub to the application.

---

## The Runtime Is a Bitstream Loader

In a conventional stack, the runtime is the layer between the language and the OS: it manages the heap, schedules goroutines or threads, runs the GC, and calls into the kernel via syscall. The runtime and the OS are distinct because they were written by different people for different purposes.

In a clock-aware stack, the distinction dissolves.

The closest analogy in hardware is a bitstream loader. An FPGA bitstream loader receives a compiled netlist — routing, LUT configuration, clock constraints, I/O assignments — and programs it onto the fabric. It does not interpret the netlist at runtime. It does not arbitrate between competing netlists. It loads the configuration, the fabric executes it, and the loader steps aside. The runtime in the clock-aware model is this exactly: it receives a compiled circuit manifest — timing windows, lifetime declarations, channel subscriptions, cryptographic identity — programs it into the dispatch table, pre-allocates the declared memory, and steps aside. The circuit executes. The runtime does not intervene again unless a signal arrives on a declared channel.

The compiler is Vivado. The manifest is the bitstream. The dispatch table is the fabric. The clock is the clock. The analogy is not approximate — it is the same model applied in software.

**The compiler and the runtime are intentionally separate.** This is not a gap in the model — it is the model's correct factoring. Developers compile on their own machines. The manifest — a signed, self-contained proof artefact — is what ships to the production server. The runtime on the production server never runs the compiler. It receives the manifest, verifies the cryptographic signature against the key ring in its `system.cap`, checks that the declared resource profile fits the production machine's declared capacity, and loads the circuit into the dispatch table. If any check fails, the manifest is rejected before a single window is allocated. The compiler is not present at runtime. It does not need to be. The proof travels with the manifest.

This separation is the same separation that makes hardware distribution work: Vivado runs on the engineer's workstation; the bitstream ships to the FPGA on the target board; the board does not need Xilinx software to run. The compiler's job ends when the proof is signed. The runtime's job begins when the proof arrives. They share nothing except the manifest format and the `system.cap` schema — and both of those are versioned, declared, and verifiable.

The roles are precisely distinct. The compiler **proves feasibility**: given this hardware model, this circuit fits, this budget holds, this L1 footprint is resident, this compliance target is met. The proof is static, complete, and produced once. The runtime **enforces it**: at every clock edge, on every core, in every tick, the dispatch table is executed and the atom stream records what actually happened. The compiler's proof is a certificate of what should happen. The runtime's atom stream is a certificate of what did happen. Both are cryptographically signed. Neither is optional.

The atom stream is therefore a continuous audit log — not a debug facility, not an optional trace, but a permanent, hardware-sourced record of every enforcement decision the runtime made. It is available at all times: to the `ObservabilityCircuit` for live monitoring, to the `AdaptationCircuit` for ML-driven optimisation, to external audit tools that subscribe to `channel AtomStream`, and to post-incident forensic analysis that can replay any tick from any point in the system's history. A regulator, a certifier, or a security auditor can verify, from the atom stream alone, that every declared timing constraint was honoured, every circuit ran under its correct key, and every handoff completed within its declared staleness bound — not as a claim, but as a cryptographically verifiable sequence of hardware-sourced measurements.

The same mechanism is directly valuable to semiconductor manufacturers for yield control. Every chip that runs a clock-aware system is continuously comparing its actual execution latencies against the declared microarchitecture model in `system.cap`. A chip that consistently executes within the declared latency bounds is verified silicon — it is meeting its speed grade in production, under real workload, not just under synthetic test vectors at the factory. A chip that shows a systematic delta — every L1 access costs 1.1 cycles instead of 1.0, every branch unit resolves in 2 cycles instead of 1 — is a chip whose actual silicon characteristics differ from its declared model. That delta is measurable, attributable, and continuous.

The consequence for manufacturers is a feedback loop that has never before existed: field measurements of actual silicon behaviour, at instruction granularity, from every deployed chip, aggregated back to the process characterisation team. Not sampled. Not estimated from test chips. From every production chip, in every deployment, every tick. A systematic shift in field measurements across a production batch identifies a process deviation before it becomes a customer-visible failure. A marginal chip that passes factory test but drifts under sustained thermal load is identified in the field, from the atom stream, before it causes an incident. The atom stream is a continuous, hardware-sourced yield monitor — and it requires no additional silicon, no test equipment, and no factory involvement. It is a consequence of running a clock-aware system.

### The Compiler Produces a Complete Static Resource Profile

When the compiler compiles a circuit it does not only produce executable code. It produces a complete, static, ahead-of-time picture of everything the circuit will ever use during its lifetime. This profile is part of the manifest — the bitstream the runtime loads.

**Port utilisation timeline.** For every channel the circuit subscribes to, the compiler knows exactly when it reads and when it writes: which tick within which window, at what frequency, producing or consuming what type. The profile is a timeline, not a set of bounds:

```
channel  channel RawMessage   read    tick 0    of every 12-tick window  (consumer)
channel  channel Price        write   tick 10   of every 12-tick window  (producer)
channel  channel HandoffLatency write tick 11   of every 12-tick window  (observability)
```

The runtime loads this profile and hands it to the observability sub-circuit, which uses it as the baseline: if a read or write does not appear at the declared tick, the deviation is itself an observable event. The port utilisation data the observability sub-circuit reports is not sampled — it is the delta between the declared timeline and the measured one.

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

## The OS

### The Runtime Is the Kernel

The kernel is a collection of timed functions. Each has a declared core, a declared cycle budget, a declared channel subscription list. The kernel's IRQ handler is a timed function. The kernel's RCU reclaim is a timed function. The kernel's memory management is a timed function. Each writes into typed channels; each reads from typed channels; none preempts any other because their windows are declared non-overlapping.

This is identical to an application function. The only distinction between a kernel function and an application function is the channel subscription list. Kernel functions subscribe to hardware-backed channels (`channel IrqSignal`, `channel TimerTick`). Application functions subscribe to logical channels (`channel NicFrame`, `channel OrderDecision`). In both cases the programmer writes the same channel API — the compiler resolves the physical backing from `system.cap`. Both are compiled with the same proc-macro, verified against the same `system.cap`, sealed into the same signed manifest.

The runtime is not a separate thing that sits between the language and the kernel. The kernel is the runtime. Both are collections of declared-timing circuits, compiled and verified by the same toolchain.

### The OS Is a Collection of Circuits

The operating system is not a special-privileged process. It is a collection of declared-timing circuits, written in the language, compiled by the same toolchain, verified by the same compiler. Apart from the ~500 lines of Assembly stubs that handle boot and hardware initialisation, every line of the OS is a function or a class with a `@Timeslice` annotation, a lifetime type, and a channel subscription list.

Put precisely: **the kernel is a circuit whose driving input is `channel CPUClockTick`.** Every other event in the system — a packet arriving, a value crossing a threshold, a circuit completing its window — is a consequence of that one signal propagating through the dispatch table. There is no other source. There is no other entry point. The clock ticks; the kernel advances; circuits execute; channels carry the results forward. The entire execution model reduces to that chain.

There is no privileged mode at the language level. The distinction between an OS circuit and an application circuit is the channel subscription list: OS circuits subscribe to hardware-backed channels (`channel IrqSignal`, `channel TimerTick`, `channel DmaCompletion`); application circuits subscribe to logical channels (`channel NicFrame`, `channel OrderDecision`). The language is the same. The compiler is the same. The four rules are the same.

### Handoff: Declared Cycle Boundaries, Not Locks

OS circuits communicate by writing to declared registers or memory locations at declared cycle boundaries. Circuit A writes at cycle N. Circuit B reads at cycle N+1. The compiler accounts for the timing of the handoff: it knows A's write cycle and B's read cycle, and proves that B's read is scheduled strictly after A's write. No synchronisation primitive is needed — the timing proof is the synchronisation.

This is not shared memory protected by a lock. Shared memory protected by a lock is a compensating mechanism for not knowing when two threads will access the memory. Declare when they access it and the lock is structurally unnecessary. The memory location is not "shared" in any meaningful sense — it is a typed channel between two declared-timing circuits, backed by a register or DRAM location chosen by the compiler from the declared lifetime tier.

### Live Observability: Dedicated Sub-Circuits

The OS contains a layer of dedicated sub-circuits whose sole function is observability. These circuits run in their own declared cycle windows, isolated from hot-path application circuits:

- **Port utilisation** — per-channel read/write counts, saturation, backpressure events
- **Subsystem registry** — which circuits are present in the current compiled configuration
- **Active/idle status** — per-circuit: currently executing, waiting at cycle boundary, or idle (not configured)

These sub-circuits do not poll. They produce `channel SystemSnapshot` at a declared low-frequency cycle (e.g. every 100 ms). The observability data is always fresh — not sampled, not aggregated by a runtime monitor that wakes up occasionally — because the sub-circuits are always running in their declared windows. A circuit that fails to reach its cycle boundary produces a gap in the snapshot stream, which is itself an observable event.

This replaces `/proc`, `/sys`, `perf_event_open`, `eBPF`, and every observability layer bolted on after the fact — because those layers exist precisely because the original system did not declare what it was doing.

### Configurable Circuit Selection

Which OS circuits execute is a build-time declaration in `system.cap`. On a bare-metal HFT system:

```
circuits.os = [ClockCircuit, NicCircuit, MemoryCircuit, ObservabilityCircuit]
```

On a development machine:

```
circuits.os = [ClockCircuit, NicCircuit, MemoryCircuit, FilesystemCircuit, DisplayCircuit, ObservabilityCircuit, DebugChannel]
```

Circuits not listed in the configuration emit no code. There is no dead-code elimination pass at link time — the circuits are simply never compiled in. A bare-metal HFT build has no filesystem code in the binary, by construction, not by convention. A circuit that subscribes to a channel not backed by any declared hardware in `system.cap` is a compile error.

### Running a Program Is Adding a Circuit

There is no process. There is no `fork`. There is no `exec`. There is no ELF loader. There is no address space switch.

Running a program means dynamically adding a compiled circuit to the live system. The circuit arrives as a signed, verified manifest — its timing declarations, its lifetime types, its channel subscriptions already compiler-verified against `system.cap`. The OS scheduler (which is the clock) examines the manifest, allocates cycle windows that do not overlap with existing circuits, pins the circuit's working set into the declared memory tier, and registers its channel subscriptions. From that moment, the circuit executes at its declared cycles alongside every other circuit in the system.

Every time the runtime adds a circuit, it adds it alongside the circuit that handles its error signal. The circuit manifest declares both as a pair — the main circuit and its error-handler circuit — and the runtime registers them atomically. The main circuit never begins execution without its error-handler already subscribed to its error channel. This is not optional and cannot be deferred: a circuit whose error-signal handlers are not compiler-verified does not compile. Adding a circuit and adding its error-handler are not two operations. They are one.

The runtime supplies a default error-handler for every signal type. The default handlers are conservative: log the signal to the observability channel, remove the circuit cleanly, notify any declared dependents via `channel DependencyLost`. A programmer who does not declare a custom handler gets this behaviour automatically — and the compiler records in the manifest which signals are handled by the default and which have an explicit override. A programmer who declares a custom error-handler circuit replaces the default for the specific signal types it covers; the remainder fall through to the default. There is no way to leave a signal type unhandled: either the programmer's handler covers it, or the default does.

```
                          ┌─────────────────────────────┐
                          │         RUNTIME              │
                          │                              │
                          │  add_circuit(manifest)       │
                          │       │                      │
                          │       ▼                      │
                          │  ┌────────────┐              │
              inputs ────►│  │   CIRCUIT  │──► outputs   │
                          │  └─────┬──────┘              │
                          │        │ channel ErrorSignal │
                          │        ▼                      │
                          │  ┌─────────────────────────┐ │
                          │  │     ERROR HANDLER        │ │
                          │  │                          │ │
                          │  │  custom  │   default     │ │
                          │  │ ─────────┼────────────── │ │
                          │  │ declared │ log + remove  │ │
                          │  │ handler  │ + notify deps │ │
                          │  └─────────────────────────┘ │
                          │                              │
                          │  ← registered atomically →   │
                          └─────────────────────────────┘
```

Both halves of the pair appear in the same manifest. The compiler verifies that every signal type the circuit can emit is covered — either by a declared custom handler or by the runtime default. The runtime installs both or neither. There is no intermediate state in which the circuit runs without its error-handler present.

The protocol for `add_circuit` is seven steps, each of which completes before the next clock edge:

```
Step 1 — WRITE TO STAGING REGION
         Write the compiled circuit image into a reserved
         staging region of the declared memory tier.

         [ circuit image ] ──► [ staging region ]
                                      │
Step 2 — SET READY FLAG               │
         Set ready_to_load = true in  │
         the staging region header.   │
                                      ▼
Step 3 — WAIT FOR CLOCK EDGE    [ ready_to_load = true ]
         The caller yields at its     │
         declared cycle boundary.     │
         The clock is already coming. │
                                      ▼
Step 4 — RUNTIME READS STAGING  [ clock edge ]
         The runtime's loader         │
         circuit fires on the         │
         same clock edge. It          │
         reads ready_to_load,         │
         reads the image.             │
                                      ▼
Step 5 — WRAP WITH ERROR HANDLER [ image validated ]
         Runtime binds the error-     │
         handler (custom if declared, │
         default otherwise) to the    │
         circuit's error channel.     │
                                      ▼
Step 6 — WRITE TO DISPATCH TABLE [ pair bound ]
         Runtime writes the           │
         circuit's memory location    │
         and cycle windows into       │
         the dispatch table.          │
                                      ▼
Step 7 — SET LIVE FLAG           [ dispatch table updated ]
         Runtime sets circuit_live    │
         = true in the table entry.   │
         The circuit executes at      │
         its next declared window.    │
         The loader circuit's flag    ◄─── runtime sees it
         is already observed by the        automatically on
         dispatch loop — no            the next table scan
         explicit notification.
```

Nothing in this protocol requires a syscall, a lock, or a shared counter. The staging region is owned by the caller until `ready_to_load` is set; owned by the runtime from that point. The dispatch table is written once by the runtime and read continuously by the dispatch loop. The only shared state is the flag — and the flag is written once, in one direction, at a clock boundary.

The consequence of this protocol is the deepest constraint in the model: **the runtime must run in sync with the system clock**. Every step in the protocol — the flag check, the image read, the error-handler binding, the dispatch table write, the live flag — is keyed to a clock edge. The loader circuit fires at a clock edge. The dispatch loop advances at a clock edge. The circuit's first window begins at a clock edge. If the runtime drifted from the system clock, the protocol would break: a flag set at cycle N might not be seen until cycle N+k, and the circuit's declared windows would no longer correspond to real hardware time.

This is not an implementation detail to be managed. It is the core invariant the model is built on — the same invariant that makes the compiler's timing proofs valid. The runtime is not a program that happens to be aware of the clock. The runtime *is* the clock, from the software's perspective. Its own execution is the reference against which all declared windows are measured. A runtime that loses clock synchronisation does not produce degraded performance — it produces incorrect behaviour, because the timing declarations that the compiler proved correct are no longer being honoured by the hardware they were proved against.

This collapses the conventional distinction between scheduler and clock. In a conventional OS, the scheduler is a software component that runs on top of the hardware clock — it reads a timer, decides who runs next, and context-switches. The clock and the scheduler are separate things; the scheduler is downstream of the clock. In the clock-aware model, **the scheduler is the CPU clock**. There is no software layer that reads the timer and decides. The dispatch table is compiled from timing declarations; the clock advances; the circuit whose window begins at this edge executes. The scheduler does not make a decision — the clock makes the decision, because the decision was already encoded into the dispatch table at compile time. The clock is the scheduler. They are the same thing.

Stopping a program means removing the circuit. Its cycle windows are reclaimed. Its declared memory tier is released at the next cycle boundary — no GC, no deallocation, no reference counting. Its channel subscriptions are unregistered. The system continues without it.

The mental model is a PCB where you can slot in and remove chips while the board is powered. The rest of the board does not pause. The clock does not stop. The other circuits do not know and do not need to know. The bus just has one fewer participant.

This has a direct consequence for correctness: a program cannot affect another program's timing unless it is explicitly connected via a declared channel. There is no shared address space to corrupt. There is no scheduler to starve. There is no heap to fragment. Two circuits running on the same machine are as isolated as two chips on separate parts of a PCB — unless a wire between them is explicitly soldered into the design.

### Booting Is Two Steps: Add the Runtime, Then Add the Kernel Circuits

Boot is not a special phase. It is two sequential applications of the same primitive, performed once at power-on.

**Step 1 — Add the runtime.** The ~500 lines of Assembly stubs perform the minimal hardware initialisation that cannot yet be expressed in the language — memory controller setup, clock configuration, cache enablement, the first stack pointer. Once that completes, the runtime itself starts: the dispatch loop, the loader circuit, the memory tier manager, the clock synchronisation mechanism. The runtime is not a circuit — it is the substrate that circuits run on. After step 1, the system is alive but empty: a running runtime with no circuits in its dispatch table.

**Step 2 — Add the kernel circuits.** The runtime immediately calls `add_circuit` with the kernel manifest — a compiled, verified set of declared-timing circuits covering the timer, the memory manager, the NIC, the namespace, the observability sub-circuits, the terminal, and whatever else `system.cap` declares. Each kernel circuit goes through the standard seven-step `add_circuit` protocol: staged, flagged, read, wrapped with its error-handler, written to the dispatch table, marked live. After step 2, the kernel circuits are running at their declared cycles. The system is in normal operation.

```
  POWER ON
     │
     ▼
  [ Assembly stubs — hardware init ]
     │
     ▼
  STEP 1: ADD RUNTIME
  ┌─────────────────────────────────┐
  │  dispatch loop                  │
  │  loader circuit                 │
  │  memory tier manager            │
  │  clock sync                     │
  │                                 │
  │  dispatch table: (empty)        │
  └──────────────┬──────────────────┘
                 │ add_circuit(kernel manifest)
                 ▼
  STEP 2: ADD KERNEL CIRCUITS
  ┌─────────────────────────────────┐
  │  dispatch table:                │
  │    ClockCircuit      [live]     │
  │    MemoryCircuit     [live]     │
  │    NicCircuit        [live]     │
  │    NamespaceCircuit  [live]     │
  │    ObservabilityCircuit [live]  │
  │    TerminalCircuit   [live] ◄── user types here
  └─────────────────────────────────┘
```

There is no "kernel mode" to exit, no "userspace" to enter, no privilege boundary to cross. The Assembly stubs ran once and will never run again. The runtime is running. The kernel circuits are circuits. Any subsequent program launch is the same `add_circuit` call — step 2, repeated. Boot just happens to be the first time step 2 runs, with the kernel manifest, on a dispatch table that was empty.

Each kernel circuit has a single, narrow responsibility:

| Circuit | Responsibility | Exposes |
|---|---|---|
| `ClockCircuit` | Reads the hardware cycle counter (TSC). Calibrates it to wall-clock time. Does not drive the scheduler — the CPU clock does that directly. | `channel Tick`, `channel WallClock` |
| `MemoryCircuit` | Pre-allocates memory regions at slot time. Manages tier promotion/demotion. Handles circuit removal and reclaim. | `channel SlotAck`, `channel RemovalSignal` |
| `NicCircuit` | DMA-paced network I/O via the NIC driver. Delivers frames to subscriber circuits at declared cycles. | `channel EthernetFrame` |
| `NamespaceCircuit` | Maps names and identifiers to `Cold` tier regions. The filesystem namespace. | `channel ColdRegion` |
| `ObservabilityCircuit` | Publishes per-circuit utilisation, timing deltas, and system snapshots at a declared low-frequency window. | `channel SystemSnapshot` |
| `TerminalCircuit` | Reads keyboard input. Resolves names in the namespace. Calls `add_circuit`. Displays output. | `channel DisplayOutput` |

`ClockCircuit` is worth dwelling on. In a conventional OS, the timer interrupt is what drives the scheduler: it fires periodically, the kernel preempts the running process, picks the next one. In the clock-aware model, the scheduler *is* the CPU clock — the dispatch table encodes who runs at which tick, and the clock advances it. `ClockCircuit` has nothing to do with scheduling. It is purely a source of time-as-a-value: circuits that need elapsed time, timeouts, or wall-clock timestamps subscribe to its channels. The scheduling mechanism and the timekeeping mechanism are completely separate, which is the correct factoring.

### Callable and Non-Callable Circuits

Not all circuits are equal from the user's perspective. The OS circuit collection includes a `TerminalCircuit` — a circuit that subscribes to `channel KeyboardInput`, parses commands, and dispatches to other circuits by name. It is the user-facing surface of the system. What it can dispatch to defines the distinction between circuit kinds:

**Callable circuits** are registered in the `NamespaceCircuit` with a public name. The terminal can look them up by name and invoke them via `add_circuit`. They produce output to a `channel DisplayOutput` that the terminal subscribes to for the duration of their execution. They are the clock-aware equivalent of POSIX commands — `ls`, `cp`, `grep`, and any user-defined program that declares itself callable. A callable circuit is a first-class citizen of the namespace: it has a name, a manifest, a declared signature, and can be composed with other callable circuits via channel piping.

**Non-callable circuits** have no public name entry in the `NamespaceCircuit`. They cannot be invoked from the terminal. Two sub-kinds:

| Sub-kind | Examples | Added by |
|---|---|---|
| System circuits | `ClockCircuit`, `MemoryCircuit`, `NicCircuit`, `NamespaceCircuit`, `ObservabilityCircuit` | Boot manifest |
| Application circuits | Trading engine, risk checker, background daemon | Programmatically via `add_circuit` from another circuit |

System circuits are the kernel. Application circuits are long-running programs added by other circuits or by the terminal's own `add_circuit` call when the user requests a background process. Neither is reachable by name from the terminal prompt — they are not callable, they are running.

```
  BOOT (step 2 complete)
   │
   ▼
  ┌──────────────────────────────────────────────┐
  │  KERNEL CIRCUITS (system circuits, non-callable) │
  │                                              │
  │  ClockCircuit   MemoryCircuit   NicCircuit   │
  │  NamespaceCircuit   ObservabilityCircuit     │
  │  TerminalCircuit ◄── user types here         │
  └──────────────┬───────────────────────────────┘
                 │ looks up name in NamespaceCircuit
                 │ calls add_circuit(manifest)
                 ▼
        ┌─────────────────────┐
        │  CALLABLE CIRCUITS  │  (POSIX equivalents + user programs)
        │                     │
        │  ls   cp   grep     │
        │  MyProgram  ...     │
        └─────────────────────┘
                 │ add_circuit (background / long-running)
                 ▼
        ┌─────────────────────┐
        │ APPLICATION CIRCUITS│  (non-callable, always running)
        │                     │
        │  TradingEngine      │
        │  RiskChecker  ...   │
        └─────────────────────┘
```

The terminal is itself a system circuit — it is not callable from itself. It is added at boot alongside the other OS circuits and runs in its own declared cycle windows, subscribing to keyboard input and display output via their respective driver circuits. There is no shell interpreter in the traditional sense: the terminal does not evaluate a scripting language. It resolves a name in the namespace, verifies the manifest against `system.cap`, and calls `add_circuit`. The "shell" is name lookup plus circuit registration. That is all it is.

A driver is a circuit with a declared channel interface. Two structural kinds:

**EventDrivenDriver** — subscribes to a hardware interrupt or DREQ assertion. Fires when the hardware signals readiness. Produces to a `channel T` that application circuits can subscribe to.

**MessageDrivenDriver** — transfers data through channels. Three orientations:

| Orientation | Description |
|---|---|
| `ReadOnly` | Consumes from `channel T`. Produces nothing. Used for write-only hardware: DAC, actuator, display. |
| `WriteOnly` | Produces to `channel T`. Consumes nothing. Used for read-only hardware: ADC, sensor, NIC RX. |
| `BothWays` | Consumes from one channel, produces to another. Full-duplex hardware: SPI, UART, Ethernet with RX+TX. |

A driver declares its channels, its timing constraints, and its DMA configuration. The compiler resolves the DMA ring buffer, the DREQ line, and the interrupt vector from the device descriptor in `system.cap`. The driver programmer writes `channel UartFrame` — the compiler wires the UART DREQ to the DMA controller and delivers completed frames to the channel at the declared cycle. No manual DMA ring setup. No interrupt registration. No `request_irq`. No `dma_alloc_coherent`.

### A Device Is a Circuit with Declared Signal Access

A device is a `circuit` — stateful, clocked, with its register file backed by the device's internal state. Its input and output ports are declared channels; its readiness is declared as a `Signal`. Three device kinds:

```
circuit ReadOnlyDevice {
    // Device produces data. Kernel reads.
    // Input: hardware DREQ signal — is data ready?
    // Output: declared channel — delivered when DREQ asserts
    val element = Byte           // concrete element type, declared per device
    val ready:   Signal          // DREQ line — compiler monitors this
    val out:     channel Frame   // concrete channel, named per device
}

circuit WriteOnlyDevice {
    // Kernel sends data. Device consumes.
    // Input: declared channel — data to send
    // Output: hardware accept signal — is the device ready to receive?
    val element = Sample         // concrete element type
    val accept:  Signal          // device's write-ready line
    val in:      channel Sample  // concrete channel
}

circuit ReadWriteDevice {
    // Full-duplex. Both directions, independent signals.
    val ready:  Signal        // DREQ: device has data for kernel
    val accept: Signal        // device ready to receive from kernel
    val in:     channel Command   // concrete: what the kernel sends
    val out:    channel Response  // concrete: what the device returns
}
```

The `Signal` is the DREQ line — a single hardware bit that answers the question "is the device ready?" for its direction. The kernel does not poll it. The compiler subscribes the driver circuit to the signal, and the DMA controller triggers a transfer the moment the signal asserts. The circuit's next window opens with the data already in the channel — no wait, no syscall, no ISR to schedule.

```
  ReadOnlyDevice (e.g. NIC RX)

  hardware ──► DREQ asserts ──► DMA transfers ──► channel EthernetFrame written
                    │
                    └── compiler-wired: driver circuit fires on next clock edge
                                        after DREQ assertion, not before

  WriteOnlyDevice (e.g. DAC, actuator)

  channel Sample written ──► accept asserts? ──► DMA transfers ──► hardware
                                    │
                                    └── compiler holds transfer until accept is high:
                                        never writes to a device that isn't ready
```

This completes the connection. The device declares its readiness via `Signal`. The compiler wires the signal to the DMA trigger. The DMA delivers data to or from the channel. The downstream circuit's window opens with the data present. Everything between DREQ assertion and circuit execution is compiler-resolved — no runtime discovery, no interrupt handler to schedule, no quiescent state to emerge from. The signal is up: data flows. The signal is down: nothing moves. The hardware tells the truth; the compiler listens at build time.

### Everything Is DMA. Everything Is Subscription-Based.

The driver never polls. The driver never blocks. There is no path from hardware I/O to CPU that goes through a polling loop or a blocking syscall. Every hardware data transfer in the OS is a DMA operation, scheduled by the compiler, paced by the hardware's own DREQ signal.

Application circuits cannot read hardware channels directly. They subscribe through the driver that owns the channel. The driver declares which `channel T` it exposes and which subscriber identities are permitted, in its subscription list in `system.cap`:

```
driver.Uart0.exposes = channel UartFrame
driver.Uart0.permits = [SerialLogger, DebugMonitor]
```

This is the permission model. Not a syscall table. Not a capability token. Not a privilege ring. A channel subscription, declared in `system.cap`, verified by the compiler at build time. A circuit that attempts to read a channel not listed in its subscription declaration is a compile error. The permission check is not a runtime gate — it is a proof obligation discharged at compile time.

### Gathering and Non-Gathering Resources

In the ARM memory model, **Gathering** means the hardware may merge multiple accesses to the same address into one bus transaction. **Non-Gathering** (device memory attribute, `nGnRnE`) means no merging, no reordering, no speculation — every access reaches the physical device register in declared order. This distinction maps directly onto the clock-aware resource model:

| Resource type | ARM attribute | Timing mechanism | Who enforces order |
|---|---|---|---|
| Normal memory, `channel T` buffers, declared tiers | Gathering | Software-timed (window boundary) | Compiler |
| Device registers, MMIO, DMA descriptor rings | Non-Gathering (`nGnRnE`) | Hardware-timed (DREQ signal) | Hardware |

**Software-timed (Gathering) resources** are scheduled by the compiler. The hardware may merge and reorder accesses within a window because the compiler has already proved the ordering is safe across window boundaries. No barrier instruction is needed — the window boundary is the ordering proof.

**Hardware-timed (Non-Gathering) resources** are driven by the DREQ signal. The compiler wires the signal to the DMA trigger at build time. The CPU never touches the hot path. Non-Gathering semantics are the default for `ReadWriteDevice` and `ReadOnlyDevice` channel mappings — the compiler emits the correct memory attribute in the physical address mapping and the DMA descriptor, so the hardware enforces ordering without any runtime instruction from the CPU.

The consequence: `dmb` / `dsb` / `isb` memory barrier instructions are absent from the hot path. A barrier is a runtime signal that the programmer could not express the ordering statically. In the clock-aware model, the ordering is declared — Gathering resources via the window boundary, Non-Gathering resources via the hardware signal. The compiler proves both. No barrier is necessary.

### Channel Subscription Inference

The programmer does not need to explicitly subscribe to every channel. Subscription is inferred from how the channel is used:

- **A channel passed as a function parameter** is subscribed by default. The compiler sees the function's declared signature, observes the `channel T` parameter, and registers the subscription in the manifest automatically. No annotation required. If the calling circuit is not in the permitted subscriber list in `system.cap`, the registration is a compile error — but the subscription intent itself does not need to be declared separately from the type.

- **A channel declared as internal state** has its role — publisher or consumer — inferred from usage. If the circuit writes to it, it is a publisher. If the circuit reads from it, it is a consumer. If the circuit does both, it is a state owner — a circuit that both produces and consumes the same channel (e.g. an accumulator or a ring buffer manager). The compiler derives the role from the declared operations, not from an annotation. A mismatch — a circuit declared to publish a channel it only reads — is a compile error.

The result is that channel wiring is a consequence of the type system, not a separate declaration layer. The programmer writes functions that take and return `channel T` values. The compiler builds the full subscription graph from the function signatures and usage patterns. The subscription graph is the routing table. It was always implicit in the code — the compiler makes it explicit and verifiable.

---

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

**WATCH — Between windows.** The runtime never truly idles. It spins continuously on the hardware counter (`CNTVCT_EL0` / `RDTSC`), reading it on every iteration of the dispatch loop. On each iteration it also scans the dispatch table for `early_fire` flags set by incoming `channel IrqSignal` writes. This scan cannot be deferred — deferring it would delay IRQ response by however long the runtime slept, defeating the nanosecond latency guarantee. The runtime has no sleep state. Between windows it uses every tick productively: issuing exact prefetches for the next circuit's declared memory footprint, pre-conditioning the `cpu_model` frequency for a declared burst, and managing cores with no assigned circuit this window — placing them in `STANDBYWFI` (which wakes on the next timer tick, not on an OS wakeup call) or pre-warming their caches. `STANDBYWFI` is not sleep in the OS sense: the core halts instruction retirement but the runtime's counter-read loop resumes the instant the next declared window tick arrives. The hardware wakes it; the runtime was already waiting at the counter.

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
    fn parse(frame: channel EthernetFrame) = ...
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
fn parse(frame: channel EthernetFrame) = {
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
    fn evaluate(samples: channel ParsePriceMeasurement) = {
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

## Memory

### The Runtime Is the Memory Manager

Memory management is determined by lifetime types, which are determined by cycle annotations. The compiler knows at build time which memory tier every value lives in and when it expires. At the cycle boundary, memory is reclaimed automatically, without a single runtime instruction dedicated to collection.

The runtime operates exclusively on **physical addresses**. There are no virtual addresses in the hot path. No page table walks. No TLB lookups. No TLB shootdowns. Every memory region used by every circuit is declared in the manifest with its physical base address and size, pre-allocated at slot time, and pinned for the duration of the circuit's declared lifetime. The runtime tracks memory handoffs — which circuit owns which range at which tick — directly in the manifest record. This eliminates indirection entirely: the compiler knows the exact cache set and cache way each physical address maps to (given the cache size and associativity declared in `system.cap`), so it can arrange memory instructions to avoid set conflicts at the instruction level, not just at the allocation level. Physical addressing makes memory instruction scheduling deterministic. Virtual addressing, with its TLB interposition, does not.

The two-question memory manager:
1. Did the function request allocation this cycle? → serve from the declared tier.
2. Did the function finish this cycle? → reclaim its entire working set.

Every GC algorithm ever invented — generational, concurrent, incremental, ZGC, Shenandoah — is a runtime approximation of this two-question circuit. The approximation exists because the GC does not know the mutator's timing. Declare the timing and the approximation becomes unnecessary.

### No Stack, No Heap — One Flat Physical Address Space

A conventional programme has two memory regions the programmer reasons about separately: the stack (implicit, automatic, scoped to call depth) and the heap (explicit, dynamic, allocated and freed on demand). Both exist because the runtime has no prior knowledge of what the programme will need — the stack grows as calls deepen, the heap grows as allocations are requested. Neither region's size is declared in advance. Both are reactive.

In the clock-aware model, neither exists as a concept. There is one flat physical address space, partitioned at compile time into declared regions, each owned by a declared circuit for a declared lifetime. The compiler produces this partition from the manifests of all slotted circuits. The runtime loads it before the first window opens. It does not change at runtime. The region boundaries are physical addresses — not virtual ranges managed through a page table, not stack frames managed through a stack pointer, but fixed locations in DRAM assigned once and pinned.

```
  Physical address space (determined at compile time from all manifests)

  0x0000_0000  ─────────────────────────────────────
               runtime tables (dispatch, manifest, key ring)
  0x0010_0000  ─────────────────────────────────────
               DMA regions — NIC RX ring, NIC TX ring
               NicCircuit writes here; no CPU involvement
  0x0020_0000  ─────────────────────────────────────
               channel EthernetFrame (PacketProcessor subscriber)
               declared size = 64 × sizeof(Frame)
  0x0030_0000  ─────────────────────────────────────
               PacketProcessor task tier
               declared size = 128B, window scope
  0x0040_0000  ─────────────────────────────────────
               OrderBook session tier
               declared size = 10MB, circuit lifetime
  ...          ─────────────────────────────────────
               cold tier — NVMe-backed, demand-loaded
  ─────────────────────────────────────────────────
```

Every device knows exactly where to write its data. The NIC's DMA descriptor ring points to the physical address of `channel EthernetFrame` — a fixed address, declared in the manifest, wired into the DMA descriptor at slot time. When the NIC's DREQ asserts, the DMA controller writes directly into that address. No CPU instruction. No interrupt. No buffer copy. The data arrives at the physical address the subscriber will read from, because that address was declared before the first packet arrived.

The subscriber — `PacketProcessor` — knows the same address. Its manifest declares `channel EthernetFrame` as an input, and the compiler resolves the physical base address at build time. When `PacketProcessor`'s window opens, its first instruction fetches from a known physical address that the hardware prefetcher has already been instructed to load — because the runtime, during the preceding IDLE tick, issued the prefetch against that exact address, knowing from the dispatch table that `PacketProcessor` would need it.

This is what makes prefetching exact rather than speculative. A conventional hardware prefetcher observes past access patterns and guesses future ones — it is correct when the access pattern is regular, wrong when it is not. The runtime knows the next circuit's entire memory footprint before that circuit's window opens, because the footprint is in the manifest. It does not guess. It reads the manifest entry for the upcoming circuit, finds the physical addresses of its declared channels and tier values, and issues prefetches against those exact addresses during the idle ticks immediately before the window opens. The prefetch is a theorem execution, not a statistical inference.

The same flat address space is what makes instruction scheduling exact. The compiler, knowing the physical address of every value a circuit will access, can arrange the circuit's instruction sequence to maximise pipeline utilisation against the specific cache set layout of those addresses. Cache set conflicts — two values that hash to the same cache set and evict each other — are detected at compile time by computing `address mod (cache_sets × cache_line_size)` for every declared value and checking for collisions. If a collision exists, the compiler adjusts the physical placement of one of the values (widening a tier region by a cache line) until the conflict is resolved. This analysis is possible only with physical addresses. Virtual addresses, with their page-table indirection, do not allow the compiler to know which cache set a value occupies.

The consequence is a complete inversion of the conventional memory model. In a conventional system, the programme declares what it wants to do and the runtime figures out where to put things. In the clock-aware model, the compiler decides where everything goes, the runtime pre-positions it, and the programme executes against a memory layout that was optimised before the first instruction ran. The programme does not manage memory. It declares what it needs. The compiler and runtime do the rest — once, at build and slot time, never again at execution time.

### The Runtime Is the Sole Memory Provider

No circuit allocates memory. No circuit calls `malloc`. No circuit holds a pointer to an allocator. The runtime is the only entity in the system that provides memory — and it does so exclusively by serving declared lifetime types from the correct tier.

When a circuit is slotted into the dispatch table, the runtime reads its manifest: every value it declares, its tier, its size, its scope. The runtime pre-allocates the circuit's entire working set from the appropriate tier before the circuit's first window opens. The circuit's first instruction executes into memory that is already present, already pinned, already at the correct cache level. There is no allocation on the hot path — ever. The allocation happened at slot time, not at execution time.

This has a precise consequence for reclaim. Because the runtime provided every byte the circuit uses, and because every byte's lifetime is declared in the manifest, the runtime knows exactly when each value expires:

| Lifetime | Reclaim trigger | Reclaim cost |
|---|---|---|
| `Register` | Expression boundary | Zero — register file reuse |
| `ephemeral` | Function return | Zero — L1 slot reuse |
| `task` | Cycle window close | Zero — L1/L2 slot reuse |
| `session` | Circuit removal | Single range deallocation |
| `permanent` | Programme termination | Single range deallocation |
| `Cold` | Declared unload cycle | NVMe page reclaim |

"Reclaim cost" is zero for the hot tiers because there is nothing to collect — the slot was pre-allocated, the window closed, the same slot is used for the next circuit's matching tier. There is no free list to walk, no reference count to decrement, no mark phase, no sweep phase. The runtime increments a pointer. The slot is reclaimed.

For `session` values — cross-circuit handoffs like `riskLimits` — reclaim happens when the owning circuit is removed from the dispatch table. The runtime reads the circuit's manifest, walks its declared `session` values, and releases each range back to the DRAM pool in a single operation. The cost is proportional to the number of declared `session` values in the manifest, not to the size of the data. The runtime never touches the data itself — it reclaims the address range. A `session<RiskLimitTable>` with 10 MB of data costs the same to reclaim as one with 10 bytes.

This is why the runtime must be the sole provider. A circuit that allocated its own memory would hold a range the runtime does not know about. The runtime could not reclaim it at circuit removal without scanning — which requires knowing where to scan, which requires the circuit to have declared it. The declaration is the allocation. The allocation is the declaration. The runtime holds both ends of the same fact.

The programmer never sees this. They declare `session<RiskLimitTable>` and write to it. The runtime provided the memory before the first write. The runtime reclaims it after the last read. The circuit never held a pointer to an allocator, never called a destructor, never decremented a reference count. The lifecycle is not managed — it is declared. The runtime executes the declaration.

### Unaligned Cache Lines — Compile-Time Padding

The compiler detects structs whose declared fields would span a cache line boundary and automatically regroups and pads them to alignment. This is not a hint or a warning — it is a compile-time transformation. The compiler reads the `cpu_model`'s cache line size from `system.cap` (64 bytes on x86/ARM, 128 bytes on some POWER configurations) and arranges struct fields so that no single value straddles two cache lines. Values that would straddle are either reordered (where the reorder is semantics-preserving for the declared lifetime type) or padded with explicit fill bytes. The padded layout is emitted into the manifest alongside the circuit's memory footprint declaration. The runtime uses the padded layout for pre-allocation and for the DMA descriptor — hardware and software see the same physical layout. A struct that arrives from a network DMA with a misaligned layout is a compile error: the incoming `channel T` type's layout must match the circuit's declared type exactly, and layout is part of the type.

### Out of Memory Is a Circuit Removal Event

When a new circuit is slotted and the runtime attempts to pre-allocate its declared working set, it may find the declared tier is full. In a conventional OS this is an out-of-memory condition — a crisis that triggers the OOM killer, which scans process tables, scores processes heuristically, kills one, hopes the kill frees enough memory, and logs the outcome after the fact.

In the clock-aware model there is no crisis. The runtime knows, before allocating a single byte, exactly which circuits currently occupy the full tier and exactly how much memory each one holds — because the runtime provided every byte to every circuit and holds the manifest record of each allocation. Out of memory is not a scan problem. It is an arithmetic problem: the new circuit needs N bytes; the tier has M bytes free; M < N; remove a circuit to recover at least N − M bytes.

The removal protocol:

1. **The runtime signals the circuit selected for removal.** The signal is delivered as a `channel RemovalSignal` write — the same mechanism as any other inter-circuit communication. The selected circuit receives the signal at its next declared window open.

2. **The circuit acknowledges and completes its current window.** It does not abort mid-execution. Its current window runs to completion — the timing guarantee is not violated by removal. At the window close, the circuit writes a `channel RemovalAck` back to the runtime.

3. **The runtime removes the circuit from the dispatch table.** Its windows are released. Its channel subscriptions are unregistered. Its entire declared working set — every `session`, every `task`, every `permanent` value in its manifest — is reclaimed in a single manifest-walk. No scan. No pointer traversal. The runtime reads the manifest entries and releases the address ranges.

4. **The cause is written to the observability log.** The removal event is a `channel CircuitEvent` write to the observability sub-circuit, containing: circuit identity, removal cause (`OutOfMemory`), tier that was exhausted, bytes reclaimed, timestamp in ticks, and the identity of the circuit whose slot request triggered the removal. The log entry is structural — it is a typed channel write, not a string printed to stderr.

The runtime selects which circuit to remove by priority, declared in `system.cap`:

```
circuits.eviction.policy = LowestPriority
circuits.eviction.exempt = [ClockCircuit, MemoryCircuit, ObservabilityCircuit]
```

Exempt circuits cannot be evicted. Priority among non-exempt circuits is declared — not heuristically scored at runtime. The eviction decision is deterministic and reproducible: given the same manifest and the same memory pressure event, the same circuit is always removed. There is no OOM killer randomness. There is no "it killed the wrong process" post-mortem.

The reclaimed bytes are immediately available for the new circuit's pre-allocation. The slot request that triggered the eviction is retried once the ack is received. If the reclaimed amount is still insufficient — because the evicted circuit was smaller than needed — the process repeats with the next lowest-priority circuit. Each eviction is a separate log entry. The sequence of removals that preceded a successful slot is fully traceable from the observability channel.

### The Filesystem Is the Cold Tier

A traditional filesystem bundles three distinct concerns: block I/O, caching, and a namespace. In the clock-aware model, two of those three are already solved by the tier system and the driver model.

**Block I/O** is the NVMe driver circuit: DMA-paced, DREQ-driven, timing declared, no polling, no blocking. It is already present in the OS circuit collection.

**Caching** is the tier system itself. Data that starts in the `Cold` tier (NVMe, demand-loaded) and is promoted to `permanent` or `session` (DRAM, pinned) *is* the cache — with the promotion policy declared at compile time rather than managed by a page replacement algorithm at runtime. There is no separate buffer cache to maintain. There is no `drop_caches`. The tier system is the cache, and the runtime is its manager.

**The namespace** — the mapping from names to locations in the `Cold` tier — is the one part that requires a dedicated circuit. This is the `NamespaceCircuit`:

```
  name / identifier
         │
         ▼
  ┌─────────────────────┐
  │   NamespaceCircuit  │
  │                     │
  │  name ──► cold T   │   reads/writes via NVMe driver circuit
  │           region    │──────────────────────────────────────►
  │           + offset  │
  │           + size    │
  └─────────────────────┘
         │
         ▼
  channel ColdRegion   (returned to caller at declared cycle)
```

The `NamespaceCircuit` maps identifiers — circuit names, handoff keys, manifest hashes, application-level paths — to declared `Cold` regions. It holds no cache of its own; the tier system already promotes hot regions into DRAM. It performs no journaling in the traditional sense; durability is the declared write-commit policy of the `Cold` tier, enforced at the clock boundary by the NVMe driver circuit. It holds no locks; lookups are channel reads, writes are channel writes, and conflicting writes to the same region are resolved at the clock boundary by the runtime's ownership model.

| Traditional filesystem concern | Clock-aware equivalent | Where it lives |
|---|---|---|
| Block I/O | NVMe driver circuit | OS circuit collection |
| Page cache / buffer cache | `permanent` / `session` tier promotion | Runtime tier manager |
| Cache eviction policy | Tier demotion on memory pressure | Runtime (declared weights) |
| Journal / durability | `Cold` write-commit at clock boundary | NVMe driver circuit |
| Namespace (paths, inodes) | `NamespaceCircuit` | OS circuit collection |
| File locking | Channel ownership at clock boundary | Runtime ownership model |
| `open` / `close` / `mmap` | `cold T` channel subscription | Language type system |

There is no `VFS` layer. There is no `inode` table separate from the namespace. There is no `dentry` cache separate from the `permanent` tier. There is no `fsync` — the write-commit cycle boundary *is* the sync. A programme that holds a `cold T` handle is subscribed to that region; when it drops the handle (removes the subscription), the runtime releases the region back to the `NamespaceCircuit`. There is no file descriptor table. There is no `open file description`. There is a typed channel subscription — the same primitive used for every other resource in the system.

The filesystem was always a cache manager bolted on top of block I/O with a namespace on top. When the cache manager is the runtime, the block I/O is the driver circuit, and the namespace is one more OS circuit, the filesystem disappears as a concept — and what remains is simpler, faster, and fully verified by the same compiler that verifies everything else.

### Atoms — The Audit and Adaptation Unit

An **atom** is the smallest observable unit of execution in the clock-aware model: one circuit, one cycle window, one tick. Every instruction executed within that window, every register touched, every channel read or written — all of it is attributed to a single atom. The trace unit (see `### The Trace Unit`) emits one `TraceEvent` per atom.

Atoms serve two purposes:

**Audit.** Every atom is a self-describing unit: it carries the circuit identity, the tick, the core, the execution cost, and whether the execution plan was honoured. A sequence of atoms is a complete, tamper-evident, instruction-level audit log of the system's execution. Forensic analysis of any production incident reduces to replaying the atom stream up to the incident tick and inspecting state at any point. There is no sampling. There is no inference. The record is complete.

**Adaptation.** Atoms are also the training signal for the runtime's ML-based execution planner. The runtime maintains a model of each circuit's actual behaviour — how often it uses its full budget, how often it finishes early, which channels it reads first, how its cost varies by input pattern. This model is updated continuously from the atom stream. The runtime uses it to make two kinds of real-time decisions:

| Decision | Basis |
|---|---|
| Tick window sizing | If a circuit consistently uses 60% of its declared budget, the runtime can tentatively compress its window and reallocate the surplus — subject to the compiler's minimum bounds |
| Core placement | If two circuits' atom streams show high channel affinity (producer/consumer in consecutive ticks), the runtime migrates them to adjacent cores to reduce handoff latency |

These are not policy overrides. The compiler's timing proof remains the hard floor — the runtime can only tighten within the declared window, never exceed the declared budget. The ML model is the runtime's way of finding slack within the proof, not of relaxing the proof.

### Hot and Cold Cores — Clock Model Assignment at Runtime

Not all cores run at the same clock model. The runtime maintains a live utilisation map of every core and assigns a `clock_model` to each based on current load:

| `clock_model` | Condition | Effect |
|---|---|---|
| `Performance` | Core is executing hot-path circuits at high utilisation | Maximum frequency, full pipeline depth, all execution ports active |
| `Balanced` | Core is in mixed use — some hot, some background | Moderate frequency, reduced port count |
| `Efficiency` | Core is executing only cold-path or background circuits | Minimum frequency sufficient for declared window — no excess power |
| `Idle` | No circuits scheduled on this core this tick | Low-power state, clock gated |

The assignment is made at clock-boundary granularity — not as a sustained mode, but as a per-tick decision derived from the dispatch table. The runtime knows, from the dispatch table, exactly which circuits will execute on each core in the next N ticks. It can pre-assign the `clock_model` for those ticks before they arrive. The core transitions power state between windows, not reactively to load spikes.

This means the CPU is running at exactly the power level the declared workload requires — never more, never less. There is no DVFS governor sampling utilisation and guessing. The dispatch table is the workload declaration. The clock model follows from it, provably.

A pipelined hot-path circuit arriving at a core that was in `Efficiency` mode triggers a pre-transition: the runtime, seeing the incoming circuit in the dispatch table, upgrades the core's `clock_model` to `Performance` one window before the circuit starts. The circuit never executes at reduced frequency. The transition is declared, not reactive.

**The asymmetry between hot-core speed and data arrival rate has a direct consequence for die design.** Hot cores process so fast — nanoseconds per circuit window — that data simply does not arrive quickly enough to keep them fully occupied. A 10 Gbps NIC delivers one 1500-byte frame every 1.2 µs. A hot core running at 4 GHz executes 4800 cycles in that window. A well-pipelined packet processing circuit might use 200 of them. The remaining 4600 cycles the hot core sits idle between bursts.

Because the compiler proves exactly how many hot-path cycles each circuit needs, the runtime knows this idle fraction precisely — not as a measurement, but as a theorem derived from the dispatch table. That idle fraction is not wasted. It is reclaimable. The runtime fills the gaps between hot-path bursts with cold-path circuits on the same core — observability, logging, namespace lookups, background maintenance — or it schedules those on dedicated cold cores running in `Efficiency` mode at a fraction of the power.

The implication for the hardware architecture is that the optimal core mix is **few hot cores, many cold cores**:

```
  Die area budget fixed
  ─────────────────────────────────────────────────────────
  Conventional CPU:  many identical cores, all capable of hot-path
                     → most cores underutilised most of the time
                     → die area wasted on capability that is rarely used

  Clock-aware CPU:   few HOT cores (Performance, full pipeline depth)
                     many COLD cores (Efficiency, simple pipeline, low power)
                     → hot cores handle bursts, proven to fit in their windows
                     → cold cores handle everything else, continuously
                     → more total processing units for the same die area
                     → more total throughput for the same power envelope
```

A conventional CPU cannot make this choice because it cannot prove which workloads are truly hot and which are cold — it must assume any core might need to run anything. The clock-aware model makes the distinction compile-time: every circuit declares its tier, its budget, its timing. The hardware can be matched to the workload structure that the compiler proved, not to a worst-case generalisation of it. Fewer hot transistors. More cold transistors. More work done. Less power consumed.

### ARM Clock Domains — Mapping to the Tick System

On ARM, the hardware exposes several distinct clock inputs that the runtime maps directly onto declared tick sources:

| ARM clock signal | Purpose | Runtime mapping |
|---|---|---|
| `CLKIN` | Core clock — the fundamental execution tick | The base of every declared `@Timeslice` window. All cycle budgets are in `CLKIN` ticks. |
| `CNTCLKEN` | Generic counter clock — drives `CNTVCT_EL0` (wall-clock counter) | The `ClockCircuit`'s `channel WallClock` source. Frequency declared in `system.cap` as `cnt_freq_hz`. |
| `ATCLKEN` | Trace clock — drives the ETM (Embedded Trace Macrocell) | The `ObservabilityCircuit`'s trace source. Must be synchronised to `CLKIN` for sub-cycle resolution. |
| `ACLKEN` | AXI bus clock — drives the interconnect | The DMA descriptor ring clock. Declared in `system.cap`; compiler verifies DMA transfer timing against AXI clock ratio. |

All four clocks are declared in `system.cap` with their frequencies and their relationships to `CLKIN`. The compiler derives tick-count equivalents for every timing annotation from these declared values. No clock frequency is assumed, guessed, or discovered at runtime. If any clock relationship is not declared, the compiler refuses to produce a binary.

**When you declare a `clock` in a programme, the runtime schedules your circuit at `CLKIN` precision.** The `clock` keyword names a hardware clock source — resolved against the declarations in `system.cap`. The compiler translates every `@Timeslice` annotation on circuits that reference that clock into an integer `CLKIN` tick count. The dispatch table stores those tick counts as absolute offsets from the system epoch. The runtime advances the dispatch table one entry at a time by reading `CNTVCT_EL0` (or `RDTSC` on x86) and comparing it to the stored tick boundary. When the counter reaches the circuit's declared tick, that circuit executes — not approximately, not within a scheduler quantum, but at the exact `CLKIN` tick the compiler proved it should.

The `clock` declaration is therefore a binding between the programme's time model and the silicon's oscillator. A circuit declared against `clock SysClock` where `SysClock` maps to `CLKIN` at 4 GHz has its windows measured in 250 ps increments — the period of one `CLKIN` tick. A circuit declared against a derived clock — a timer running at `CLKIN / 8` — has its windows in 2 ns increments. Both are exact. Both are enforced by the same counter read. The precision is not the scheduler's precision. It is the oscillator's precision.

This is what distinguishes declared timing from OS scheduling. A Linux `SCHED_DEADLINE` task with a 4 ns period is not scheduled at 4 ns precision — the scheduler itself has microsecond granularity, cache warm-up costs, context-switch overhead, and kernel execution time that all come out of the task's budget invisibly. A clock-aware circuit with a 4 ns window is scheduled at `CLKIN` precision because the runtime does not context-switch, does not have a scheduler quantum, and has already proved that its own overhead is zero on the hot path. The `clock` declaration is honoured at the oscillator, not approximated by a software layer above it.

### Accelerator Coherency Port — Shared Memory Without CPU Involvement

The ARM Accelerator Coherency Port (ACP) allows an accelerator — a GPU ALU array, a matmul unit, a DMA engine — to make coherent requests directly to the CPU's cache without involving the CPU. The accelerator reads or writes a cache line that the CPU has declared in its manifest, and the coherency protocol ensures the CPU sees a consistent view without a barrier instruction or a CPU-side flush.

In the clock-aware model, ACP is the natural backing for cross-accelerator `channel T` handoffs where the producer is a CPU circuit and the consumer is a declared accelerator circuit (or vice versa). The compiler, knowing both the CPU window and the accelerator's declared access window from `system.cap`, inserts the correct ACP coherency gap — derived from the ACP latency for the declared SoC — as a mandatory window-boundary constraint. The accelerator reads coherently from the CPU's L2, the CPU reads coherently from the accelerator's write buffer, and no software coherency management is needed. The handoff is proven at compile time. The ACP is the physical mechanism the runtime uses to execute that proof.

### Core Power Modes — Runtime Lifecycle

ARM cores expose a structured set of power modes. The runtime manages these as a first-class part of the dispatch loop, not as a side effect of OS idle logic:

| Power state | ARM terminology | Runtime trigger | When used |
|---|---|---|---|
| On | Normal | Any circuit assigned to this core | Hot-path and cold-path execution |
| Standby | `STANDBYWFI` / `STANDBYWFI2` | No circuit assigned this window; wake event expected soon | Short idle gaps between windows; core wakes on next declared window tick |
| Retention | `Ret` | No circuit assigned for N ticks; L1 state must be preserved | Gaps longer than a few windows where re-powering L1 would cost more than retention |
| Dormant | Individual Core Shutdown | Core has no assigned circuits for the foreseeable dispatch horizon | Cold cores with no scheduled work; full power removal except for wake logic |
| Off | Full off | Core removed from `system.cap` at runtime | permanent removal; state not preserved |

`STANDBYWFI` (and `STANDBYWFI2` on multi-threaded cores) is the most common idle state. The runtime enters it at the end of a window when the dispatch table shows no circuit assigned for the immediately following window. The core halts execution and waits for an interrupt — in the clock-aware model, the only interrupt that fires is the hardware timer tick at the next declared window boundary. No spurious wakeup, no IRQ to classify, no scheduler to re-enter. The core wakes exactly when the next declared circuit is due, executes it, and returns to standby.

`Dormant` mode is used for cold cores that the dispatch table shows unoccupied for many ticks ahead. The runtime transitions them to Dormant via the power controller declared in `system.cap`. The wake latency for Dormant is longer than for Standby — typically hundreds of ticks — so the runtime's lookahead must see a circuit arriving with enough lead time to wake the core before the window opens. This wake latency is declared in `system.cap` and the compiler accounts for it in the dispatch table: a cold core assigned a circuit must be woken at `window_start − wake_latency_ticks`. If the lookahead is insufficient, the compiler selects a different core or adjusts the window.

The consequence is that the runtime's power consumption tracks the declared workload exactly. An idle system consumes only the power of the `ClockCircuit` and `ObservabilityCircuit` — everything else is in Standby or Dormant. A burst of hot-path circuits brings the required hot cores to On and leaves the cold cores in Dormant. The transition is provable from the dispatch table, not measured after the fact.

**Tick-by-tick regulation from dispatch table lookahead.** Every decision the runtime makes about power, cache, and frequency is derived from reading the dispatch table forward in time — not from sampling current conditions and reacting. Because the table is a compile-time static structure, the runtime always knows exactly what will execute N ticks from now. This converts every management action from reactive to predictive:

| Management action | Conventional OS | Clock-aware runtime |
|---|---|---|
| Core power mode transition | Triggered by load measurement after the fact | Issued exactly `wake_latency_ticks` before the circuit that needs the core |
| Cache pre-warm / prefetch | Hardware prefetcher guesses from past access patterns | Issued during WATCH at the exact tick the manifest says the next circuit's footprint should be in L1 |
| Frequency pre-conditioning | DVFS governor samples utilisation and ramps up reactively, often too late | Issued one window before the burst the dispatch table shows arriving |
| `early_fire` IRQ promotion | Scheduler runs, classifies interrupt, context-switches | Flag checked on every WATCH iteration; promoted circuit executes at next window boundary |

The dispatch table is therefore not just a schedule — it is the runtime's complete view of the future. Every tick the runtime spends in WATCH is a tick it uses to read that future and act on it precisely. A conventional OS operates in the present, reacting to events as they arrive. The clock-aware runtime operates in the declared future, preparing for events before they arrive. The difference is the dispatch table: a static proof of what will happen, available to the runtime at every tick, consulted on every loop iteration.

**The channel graph gives the runtime causal foreknowledge — not just temporal.** The dispatch table answers *when*. The channel graph answers *why* and *what follows*. Because every data dependency in the system is a declared channel connection — there are no other data paths — the compiler produces a complete causal graph alongside the dispatch table: circuit A produces to `channel Price` → circuit B consumes `channel Price`, produces to `channel OrderBook` → circuit C consumes `channel OrderBook`. Every link in that chain is known before tick 0. The runtime holds both structures simultaneously:

```
  Temporal foreknowledge (dispatch table):
  tick 0:    NicCircuit         fires
  tick 6:    parsePrice         fires
  tick 18:   updateBook         fires
  tick 28:   riskCheck          fires

  Causal foreknowledge (channel graph):
  NicCircuit    → channel EthernetFrame → parsePrice
  parsePrice    → channel Price         → updateBook, riskCheck
  updateBook    → channel OrderBook     → emitQuote
```

Together these tell the runtime something no conventional OS can know: not just that `updateBook` will run at tick 18, but that it will run *because* `parsePrice` produced a value to `channel Price` at tick 6, and that its output will be consumed by `emitQuote` at tick 28. The full instruction chain — from NIC frame arrival to quote emission — is declared, causal, and known in its entirety before any packet arrives.

This is what makes the prefetching exact rather than speculative. The runtime does not prefetch `updateBook`'s working set because it guesses `updateBook` might run soon. It prefetches it because the channel graph proves that `parsePrice`'s output will flow to `updateBook`, and the dispatch table proves `updateBook` opens at tick 18. Both facts are compile-time theorems. The prefetch is their logical consequence, issued at the exact tick the manifest says it must be issued to be ready. The runtime is not predicting. It is executing a proof.

**The channel graph proves the maximum safe low-power duration.** Because all data flow is through declared channels, and because every `early_fire` promotion is triggered by a write to a declared `channel IrqSignal`, the runtime can derive from the channel graph exactly which circuits are producers of `IrqSignal`-class channels and when their next window opens. If no such producer is scheduled before tick N, the runtime has a compile-time proof that no `early_fire` can arrive before tick N. It can therefore enter Dormant mode — full power removal — until `tick N - wake_latency_ticks`, knowing with certainty it will miss nothing.

This collapses the conventional trade-off between responsiveness and power. A conventional OS must choose: stay in a shallow sleep state (fast to wake, high idle power) or enter deep sleep (low power, slow to wake, risks missing an interrupt). It cannot know how long until the next real event, so it guesses. The clock-aware runtime does not guess. It reads the channel graph, finds the next `IrqSignal`-capable producer, subtracts the wake latency for the declared power state, and enters Dormant for exactly that duration. No tick of unnecessary power consumption. No risk of a missed event.

The CPU keeps itself maximally busy during declared busy periods — the dispatch table shows every instruction chain that will execute, the channel graph shows what data will flow through it, and the runtime pre-positions everything. Then, when the causal chain is exhausted and the next event is provably distant, it powers down to the minimum state that can still wake in time. The two structures together — temporal and causal — give the runtime complete knowledge of both when to work and when to rest, at tick-level precision, derived from declarations that were made before the first instruction executed.

**The hot path may never touch DRAM.** With the full dependency graph and execution plan known at compile time, the compiler can prove that the entire hot-path pipeline fits in registers and L1 cache — and if it does, DRAM is structurally unreachable on that path. Not avoided by luck. Not avoided by careful programmer discipline. Unreachable by theorem.

The proof is straightforward. For each value on the hot path, the compiler knows its declared lifetime tier (`register`, `ephemeral`, `task`) and its size. It knows L1 capacity and associativity from `system.cap`. It computes the total working set of the hot-path pipeline — every value that any circuit in the chain touches, from the first circuit to the last — and checks whether it fits:

```
  Hot path:  NicCircuit → parsePrice → updateBook → emitQuote

  Working set per circuit:
  NicCircuit:   channel EthernetFrame  (task tier, 1500B)   — arrives via DMA into L1
  parsePrice:   channel Price          (register lifetime)  — stays in register file
  updateBook:   channel OrderBook      (task tier, 128B)    — L1 pinned
  emitQuote:    channel Quote          (register lifetime)  — register file → DMA out

  Total L1 footprint: 1628B across four circuits
  L1 capacity: 32KB (from system.cap)
  Fits: yes — by 30KB
  DRAM load count on hot path: 0 (proved)
```

The DMA delivers the incoming frame directly into `channel EthernetFrame`'s declared L1-pinned physical address. The first instruction of `parsePrice` reads from L1. Its output fits in a register. `updateBook` reads from the register, writes its result to an L1-pinned task-tier buffer. `emitQuote` reads from that buffer — still in L1 — and the DMA carries the result out. The CPU never issued a DRAM load. The memory bus was idle for the entire hot-path chain.

The compiler detects when this proof holds and emits it into the manifest as a guarantee: `hot_path_dram_loads: 0`. The `ObservabilityCircuit` verifies it at runtime via the trace unit — a DRAM access on the hot path is a trace event, and a trace event on a path declared to have zero DRAM loads is an `ExecutionPlanViolation`. The guarantee is not aspirational. It is checked, continuously, in production.

When the hot-path working set does not fit in L1 — a larger order book, a wider price table — the compiler falls back to L2, then declares which values must be demoted to `session` tier (DRAM). The programmer sees the compile-time footprint report. They can choose to split the circuit, reduce the working set, or accept the DRAM latency and widen the budget accordingly. The decision is explicit, informed by exact numbers, made before the system ships.

### The Runtime Adapts — AI-Regulated OS

The atom stream, the ML execution planner, and the clock model assignment together form a system that adapts in real time to the actual workload — not by guessing, not by sampling, but by reading a hardware-sourced proof stream and acting on it within the constraints of the compiler's theorems.

The logical conclusion of this architecture is that the OS itself can be regulated by an AI circuit: a declared `AdaptationCircuit` that subscribes to `channel AtomStream`, runs inference over the execution history, and emits `channel RuntimeDirective` — window size adjustments, core placement recommendations, clock model pre-assignments — that the runtime applies at the next dispatch boundary.

This is not an AI that controls the OS in an unconstrained way. It is a circuit like any other circuit: declared timing, declared channels, compiler-verified. Its directives are bounded by the compiler's hard floor. It cannot make the system unsafe — the compiler already proved safety. It can only find and exploit slack that the static proof left on the table, because the static proof is conservative by necessity and the AI has the runtime's live data.

The result is an OS that is simultaneously formally verified (by the compiler) and continuously optimised (by the AI circuit) — not a trade-off between the two, but both at once, at different layers.

---

## Signals, Not Exceptions

### There Are No Exceptions

The OOM removal protocol is not a special case. It is an instance of the general model: every error condition in the system is a signal delivered to a circuit via a declared channel. There are no exceptions. There is no stack unwinding. There is no `try`/`catch`. There is no `panic`. There is no `SIGSEGV`. There is no `SIGKILL`.

An exception in a conventional system is what happens when a condition arises that was not declared: a null pointer, a divide-by-zero, an out-of-bounds access, a failed allocation. The language did not declare the condition; the hardware or the OS detects it at runtime; control is transferred by force through an unwinding mechanism to wherever the programmer hoped they remembered to put a handler. The exception is the runtime's way of compensating for a declaration that was never made.

In the clock-aware model, every condition that a circuit can encounter must be declared in its exhaustive match. A condition that the programmer did not handle is a compile error — not a runtime exception. By the time the circuit executes, every reachable state has an explicitly declared handler. There is nothing left for the runtime to intercept.

Error conditions that originate outside the circuit — hardware faults, resource pressure, dependency failures — arrive as signals on declared channels:

| Condition | Signal | Delivered via |
|---|---|---|
| Out of memory | `channel RemovalSignal` | Runtime → circuit |
| Dependency circuit removed | `channel DependencyLost` | Runtime → subscriber circuits |
| Budget overrun detected | `channel BudgetViolation` | Observability sub-circuit → declared handler |
| Hardware fault | `channel HardwareFault` | Driver circuit → subscriber circuits |
| Handoff staleness exceeded | `channel StalenessViolation` | Runtime → declared handler |

Each signal is a typed channel write. The receiving circuit handles it in its next window via its declared exhaustive match — the same mechanism it uses for any other input. There is no separate error-handling path. There is no control-flow transfer outside the circuit's declared windows. The circuit reads the signal, matches it, executes the declared handler, and continues — or acknowledges removal and exits cleanly.

The consequence is that error handling is visible, declared, and compiler-verified in exactly the same way as the happy path. A circuit that subscribes to `channel RemovalSignal` but does not handle it in its match is a compile error. A circuit that handles `DependencyLost` with `ignore` has explicitly declared that it tolerates losing its dependency — and the compiler records that declaration in the manifest. There are no silent failure modes. There are no swallowed exceptions. There are no unhandled panics that terminate the process and leave the system in an unknown state.

A circuit that exceeds its declared budget is not killed. The budget overrun is written to the observability channel. The declared `BudgetViolation` handler runs — which may log the event, reduce the circuit's load, or signal a downstream circuit to shed work. The system adapts through declared channels, not through forced termination.

The signal model unifies what other systems treat as three separate concerns — exceptions, signals, and inter-process communication — into one: a typed channel write, handled by an exhaustive match, at a declared cycle. One mechanism. No exceptions.

An exception is an imperative pattern. It is the runtime seizing control from the programme because the programme did not declare what to do. It belongs to the same family as the scheduler, the GC, the OOM killer, and the lock: compensating machinery that exists because something was never declared. Declare it and the machinery is unnecessary. The clock-aware model does not improve exception handling — it makes the condition that requires it structurally impossible to create.

---

## Services and Security

### Microservices Are Circuits with Declared Handoffs

This model scales directly to what is conventionally called a microservice architecture — except without the network, without the serialisation, without the service mesh, without the latency. A microservice is a circuit. Its interface is its declared `channel T` subscriptions. Its deployment is adding it to the live system. Its communication with other services is a memory handoff at a declared tick boundary.

The handoff tier determines what kind of inter-service communication you get:

| Handoff tier | Physical backing | Latency | Use |
|---|---|---|---|
| `Register` | CPU register file | 0 ticks | Same-core consecutive circuits only |
| `ephemeral` | L1 cache, pinned | 1–4 ticks | Same-core, tight pipeline |
| `task` | L1/L2, pinned | 4–12 ticks | Same-core, wider window |
| `session` | DRAM, resident | 50–200 ticks | Cross-core, long-lived shared state |
| `permanent` | DRAM, pinned, never paged | 50–200 ticks | Cross-circuit shared tables, reference data |
| `Cold` | NVMe, demand-loaded | declared cycle | Infrequent bulk data |

A trading engine and a risk checker on adjacent cores hand off a `session<RiskLimit>` table: the engine writes updated limits at the end of its window; the risk checker reads them at the start of its window on the next core. No HTTP. No gRPC. No message broker. No serialisation. A memory location, a declared write tick, a declared read tick, and a compiler proof that the read comes after the write.

The programmer can annotate handoff points to give the compiler and runtime additional information:

```java
@Handoff(tier = session, from = "TradingEngine", to = "RiskChecker", maxStaleness = "1ms")
val riskLimits: session<RiskLimitTable>
```

Circuit diagram — two clocked blocks, one wire, one declared memory tier:

```
  core 1                                          core 2
  ┌─────────────────────────────┐                ┌─────────────────────────────┐
  │        TradingEngine        │                │         RiskChecker         │
  │   @Timeslice(cycle = 10ns)  │                │   @Timeslice(cycle = 8ns)   │
  │                             │                │                             │
  │  ┌──────┐                   │                │                   ┌───────┐ │
  │  │ ALU  │                   │                │                   │  ALU  │ │
  │  │trade ├──► write at       │                │    read at ◄──────┤ risk  │ │
  │  │logic │    window_end     │                │    window_start   │ check │ │
  │  └──────┘         │         │                │         ▲         └───────┘ │
  │                   │         │                │         │                   │
  │             ┌─────▼──────────────────────────────────┐│                   │
  │             │          session<RiskLimitTable>        ││                   │
  │             │          (DRAM, pinned, resident)       ││                   │
  │             │                                         ││                   │
  │             │  @Handoff(from = TradingEngine,         ││                   │
  │             │           to   = RiskChecker,           ││                   │
  │             │           maxStaleness = 1ms)           ││                   │
  │             └─────────────────────────────────────────┘│                   │
  └─────────────────────────────┘                └─────────────────────────────┘
        │                                                        │
  tick  0────────────────10ns                    tick  10ns+gap──────────18ns+gap
        write_window_end ─────────────────────► read_window_start
                          coherency gap (compiler-computed from system.cap cache topology)
```

Compiler derivation table — what the compiler proves from this single annotation:

| Property | Derived from | Compiler action |
|---|---|---|
| Producer identity | `from = "TradingEngine"` | Verifies `TradingEngine` writes `riskLimits` in its declared window. Compile error if not. |
| Consumer identity | `to = "RiskChecker"` | Verifies `RiskChecker` reads `riskLimits` in its declared window. Compile error if not. |
| Memory tier | `tier = session` | Pins `riskLimits` in DRAM for programme run lifetime. Verifies declared size fits tier. |
| Staleness bound | `maxStaleness = "1ms"` | Translates to tick count (e.g. 3,000,000 ticks at 3 GHz). Verifies `TradingEngine`'s write period ≤ that count. Compile error if not. |
| Coherency gap | `system.cap` cache topology | Reads L3 domain membership for core 1 and core 2. Inserts mandatory gap = cross-L3 coherency latency (e.g. 60–80 ticks) between write window end and read window start. |
| Core placement | producer + consumer pair | Attempts to place `TradingEngine` and `RiskChecker` on cores sharing an L3 domain. If unavailable, uses next-best topology and recalculates gap. |
| Staleness logging | `maxStaleness` declared | Emits a `RDTSC` delta instruction at `RiskChecker`'s `riskLimits.get()` call site. Writes delta to `channel HandoffLatency` for the observability sub-circuit. |
| Anonymity prevention | both sides named | A value annotated `@Handoff` with no matching write in `from` or no matching read in `to` is a compile error. The memory location cannot be unowned. |

`@Handoff` annotations do three things:

1. **Name the producer and consumer** — the compiler verifies that `TradingEngine` does in fact write to `riskLimits` in its declared window, and that `RiskChecker` does in fact read from it. A mismatch is a compile error. Anonymous shared memory — a location written by nobody-in-particular and read by nobody-in-particular — cannot be expressed.

2. **Declare staleness tolerance** — `maxStaleness = "1ms"` tells the compiler that `RiskChecker` tolerates reading a value up to 1 ms old. The compiler translates 1 ms to tick count, verifies the gap between `TradingEngine`'s write window and `RiskChecker`'s read window is ≤ that count, and emits a compile error if `TradingEngine` might not update within the tolerance. The runtime logs actual staleness to the observability channel on every read.

3. **Enable cross-circuit optimisation** — knowing both producer and consumer, the compiler can place their windows on cores that share a cache domain, reducing coherency latency. Without the annotation, the compiler places windows by availability. With it, the compiler optimises placement specifically for that handoff pair.

`@Handoff` is not required — a `channel T` subscription already declares the connection. `@Handoff` is the programmer's way of asserting stronger properties — named ownership, staleness bounds, placement hints — that the compiler then verifies. It is an annotation that adds proof obligations, not one that relaxes them.

### Cryptographic Circuit Identity

Every compiled circuit manifest is signed with the organisation's private key before deployment. The signature covers the circuit's declared timing, lifetime types, channel subscriptions, and `@Handoff` declarations — the complete proof payload. The runtime verifies the signature before slotting the circuit into the dispatch table. An unsigned manifest, or one signed by an unknown key, is rejected at the dispatch boundary before any window is allocated, before any channel subscription is registered.

`@Handoff` extends this naturally. A handoff between two circuits is only permitted if both manifests are signed by keys within the same declared allocation:

```java
@Handoff(tier = session, from = "TradingEngine", to = "RiskChecker",
         maxStaleness = "1ms", org = "AcmeTradingCo")
val riskLimits: session<RiskLimitTable>
```

The `org` field names the allocation. At compile time, the compiler verifies that both `TradingEngine` and `RiskChecker` are signed by a key in the `AcmeTradingCo` key ring declared in `system.cap`. At runtime, the dispatch table enforces it: if a circuit with a different organisational signature attempts to read `riskLimits`, the subscription check fails — not as a runtime exception, not as an access-control list lookup, but as a structural impossibility. The memory location is not addressable from outside the declared allocation.

The consequence is that the trust boundary between organisations is enforced at the same level as the timing boundary: by the compiler and the dispatch table, not by a kernel permission check, not by a firewall rule, not by a service mesh policy. Two circuits from different organisations running on the same machine are as isolated as if they were on different machines — their memory locations are in disjoint allocations, their channel subscriptions are in disjoint key rings, and no wire between them can be declared without both sides presenting matching organisational signatures.

This replaces container namespaces, cgroup isolation, SELinux policy, and inter-process privilege separation for the clock-aware model. Those mechanisms exist because processes share a kernel and an address space and must be prevented at runtime from accessing each other. Clock-aware circuits have no shared address space — they have declared memory tiers with named owners. There is nothing to prevent at runtime because the connection was never expressible in the first place.

### The Cryptographic Proof Chain

The manifest signatures form a **proof chain** analogous to an SSL certificate chain: each signed manifest is a certificate of correctness, signed by a known key, verified by the runtime before acceptance.

```
  Root of Trust (system.cap key ring)
         │
         ▼
  Compiler signing key  ──signs──►  circuit manifest
                                         │
                                         │  contains
                                         ▼
                               timing proof
                               memory proof
                               channel proof
                               compliance proof  (if declared)
                               org key
                                         │
                                         ▼
                               Runtime verifies signature
                               against system.cap key ring
                                         │
                          ┌──────────────┴──────────────┐
                          │                             │
                    key known                      key unknown
                    chain valid                    or chain broken
                          │                             │
                          ▼                             ▼
                   circuit slotted              rejected at dispatch
                   into dispatch table          boundary — no window
                                                allocated, no channel
                                                registered, no trace
```

The runtime maintains a **known key set** declared in `system.cap`. Only circuits signed by keys in this set can run on the system. There is no runtime key negotiation, no certificate authority to contact, no OCSP check. The key ring is compiled into the system configuration at build time. Adding a new allowed key requires a new `system.cap` build — which is itself a compiled, signed artefact.

The consequence is that the set of circuits permitted to execute on a given machine is a compile-time declaration, not a runtime policy. A circuit that is not signed by a key in `system.cap` cannot execute — not because a runtime check blocked it, but because the dispatch table has no slot for an unverified manifest. The attack surface is the key ring declaration, not the runtime policy engine.

This extends the timing proof chain to a trust proof chain. The compiler proves timing. The signing key proves provenance. The runtime verifies both before admitting any circuit. The system's security posture is as strong as its weakest proof — and both proofs are compile-time artefacts.

### The System Runs in permanent Safe Mode

Because every circuit that executes on the machine is cryptographically signed, the runtime operates in a state of continuous, verified trust. This has a consequence that is easy to understate: **every hardware security mechanism that exists to compensate for untrusted execution becomes unnecessary.**

Privilege rings (ring 0 / ring 3) exist because the kernel cannot trust userspace code — it must prevent it from issuing privileged instructions. In the clock-aware model, every circuit that reaches the dispatch table was signed by a key in `system.cap`. Its instruction sequence is the exact sequence the compiler produced and the signing key certified. There is no untrusted userspace. Every circuit runs with the same trust level — full — because trust was established at compile time, not enforced at runtime through privilege separation.

Spectre and Meltdown mitigations — retpoline, KPTI, IBRS, STIBP — exist because speculative execution crosses trust boundaries: the CPU speculates across a privilege boundary and leaks information through a timing side channel. There are no privilege boundaries in the clock-aware model. There is no speculative execution across boundaries because there are no boundaries. The mitigations address a problem that cannot be stated in this model.

SMEP, SMAP, NX bits, W^X enforcement — these exist to prevent untrusted code from executing in privileged memory or privileged code from being tricked into executing attacker-controlled data. Every memory range in the clock-aware model is declared in a circuit manifest, pre-allocated by the runtime, and owned by a named, signed circuit. No range is writable and executable. No range is accessible from outside its declared owner. The address space is not partitioned by kernel policy — it is partitioned by the manifest, at compile time, before the system boots.

ASLR exists to make memory layout unpredictable to an attacker who has already achieved code execution. In the clock-aware model, code execution requires a signed manifest. An attacker without the signing key cannot introduce a circuit. An attacker with the signing key is not an attacker — they are an authorised developer. ASLR randomises a layout that was never secret from the compiler; the clock-aware model makes the layout a compile-time theorem that is only ever executed by circuits that were proven correct before they ran.

The system does not disable these mechanisms by policy. It renders them structurally irrelevant. The attack surface they address — untrusted code reaching the CPU — cannot be expressed in the clock-aware model. The signing key is the gate. The compiler is the certifier. The dispatch table is the enforcer. Everything that runs was already proven safe before it ran.

**The invariant the compiler maintains:**

> Every circuit starts exactly at its declared tick. Every circuit finishes before or at its budget. Every handoff is proven to complete before the reader's window opens. These are theorems, not assumptions.

The hardware clock is the only synchronisation primitive in the entire system. It was always there. It was always running. No OS ever listened to it.

---

## Hardware Backing

### The Runtime Resolves Physical Backing — The Programmer Does Not

The programmer writes `channel NicFrame` and calls `in.read()`. There is no DMA in the standard library. There is no `mmap`, no `ioctl`, no driver call, no interrupt registration. The channel is a channel.

The physical mechanism that backs it — DMA ring, interrupt delivery, shared ring buffer, inter-core message — is determined entirely by the subscription type in `system.cap`. The compiler reads the subscription declaration, resolves the physical backing for that channel name, and emits the correct access pattern. The programmer never wrote hardware code. The hardware was never invisible — it was declared once, in the configuration, and the compiler applied it everywhere.

This is the key distinction from conventional hardware abstraction layers. An HAL hides the hardware behind a generic interface and calls through to the driver at runtime. The clock-aware model declares the hardware in `system.cap` and the compiler proves at build time that the access pattern is correct for that hardware. There is no runtime dispatch. There is no driver. The channel type and the subscription declaration are the complete interface.

### The Peripheral Was Already Declaring Its Timing

The deepest insight is that this is not new. Hardware peripherals have always declared their timing — in silicon. The driver writer just was not listening.

Consider a DMA controller with a DREQ (DMA Request) line. The peripheral asserts DREQ exactly when it is ready for a transfer. Power is consumed only when DREQ is asserted, only when the peripheral is ready, only for exactly one transfer unit. Not polling — zero power between transfers. Not interrupt-driven — zero CPU power. Not scheduler-driven — zero context switch. The peripheral declares its timing in hardware. The DMA controller listens. The transfer happens at exactly the rate the hardware needs.

Before driver writers understood this:

```java
while (!peripheralReady()) {}   // polling — burns power
triggerDma();                    // manual, error-prone
while (!dmaDone()) {}           // more polling
```

After understanding DREQ: set the PERMAP register to the peripheral's DREQ number, arm the DMA once, and the hardware paces itself forever. Zero CPU involvement on the hot path. Hundreds of lines of driver code replaced by one 5-bit field in a register — because the peripheral was already declaring its timing. The driver writer just was not listening to it.

On ARM, the runtime can issue `PRFM` (Prefetch Memory) and `RPRFM` (Range Prefetch Memory, SVE2) instructions to pre-warm a circuit's working set before its window opens. During an IDLE tick — when the current core has no circuit assigned — the runtime uses the dispatch table lookahead to identify which circuit fires next and issues a range prefetch for its declared `task`-tier working set. On the hot path, this is redundant: the speculative pre-conditioning mechanism already ensures the working set is in L1 before the window opens. `RPRFM` is most valuable for the **first dispatch of a cold circuit** that has not yet had its working set promoted: the runtime issues the prefetch during the preceding idle window, so the circuit's first execution does not pay a cold-cache penalty. After the first execution, the working set is pinned in L1 and prefetch instructions are unnecessary — the data is already there, declared to be there, proven to be there.

The clock-aware channel model makes this structural. An SPI sensor becomes:

```java
val sensor = Channel.from(Device.SPI0, clock: 1.MHz5)
sensor.subscribe(reading -> process(reading))
```

The compiler resolves from the `Device.SPI0` descriptor: the DREQ line number, the DMA channel, the clock divider from the declared rate and `cpu_model`, the CPOL/CPHA mode, the chip-select timing, the bus address translation, the cache flush requirements, and the IRQ registration. The programmer declared what they needed — a 1.5 MHz SPI channel from device SPI0. The compiler translated that declaration into every hardware detail the runtime needs. No oscilloscope. No logic analyser. No three days debugging CPOL/CPHA mode mismatch. The device descriptor carries the mode; the compiler applies it; the compiler verifies the type matches the SPI frame size. If it compiles, the wiring is correct.

### The Runtime Is the Type Enforcer

Hardware-incorrect code does not compile. Not because a linter ran after the fact. Because the type system encodes the hardware model: `channel T` is typed, so reading the wrong type is a type error. Subscription lists are compile-time verified, so reading an undeclared channel is a compile error. Cycle budgets are verified by the proc-macro against `llvm-mca`, so exceeding the budget is a compile error. The type system is the hardware contract. The compiler is the enforcer. The runtime confirms what the compiler already proved.

### Generation 1 to Generation 2 — Closing the Loop

Generation 1 ships a working system: the language, the compiler, the runtime, the kernel circuits — all written in the language — plus ~500 lines of Assembly stubs for boot and hardware initialisation. The Assembly is not hidden; it is the declared boundary between what the language can express and what requires raw instruction sequences.

Generation 2 erases that boundary. The compiler, now running on Generation 1, rewrites the Assembly stubs as declared `channel T` functions. The boot sequence becomes a declared function with a declared timing contract. Hardware initialisation becomes a sequence of channel writes. The ~500 lines of Assembly become zero. The system is fully expressed in the language that proved hardware-correctness of everything else.

A language that can express the kernel can express its own compiler. A compiler that can prove hardware-correctness of timed functions can prove hardware-correctness of itself. The compiler and the kernel become the same artefact: a collection of declared-timing functions, verified and self-validating.

This is what it means for the runtime to be the kernel: not that the kernel is simplified, but that the distinction between compiler and kernel, between language and OS, between proof and execution — collapses.

---

## The Native Substrate for Machine Learning

Every ML-driven infrastructure system today operates on approximations. The metrics are sampled. The traces are incomplete. The causal relationships between actions and effects are inferred, not known. The model learns to predict what the system will do next because it cannot read what the system is actually doing now. This is not a limitation of the ML models — it is a limitation of the systems they are trying to observe. The ground truth was never available.

The clock-aware model makes ground truth available, completely, at cycle resolution.

The atom stream is a perfect dataset: every circuit, every tick, every instruction committed, every register touched, every cache line accessed, every channel read and written — attributed, timestamped, and signed. There is no sampling. There is no inference. The record is complete by construction. An ML model trained on the atom stream is trained on reality, not on a statistical approximation of reality.

The execution plan is the ground truth label. The compiler produces a prediction — the expected sequence of instruction timings, port utilisations, and register states — before the circuit ever executes. The atom stream is the measured outcome. The delta between them is a precise, cycle-accurate error signal. No other system in existence produces a labelled dataset this precise, continuously, from production hardware, with zero instrumentation overhead.

The dispatch table is the action space. Every decision the runtime makes — clock model assignment, core placement, L1 promotion, window compression — is an action taken on a fully declared, fully observable state. The state is the manifest. The constraints are the compiler's proofs. The ML model does not explore an unknown environment; it optimises within a completely specified one. Its actions are bounded by theorems, not by hope.

The consequence is that the clock-aware model is the first system where ML can close the loop completely:

```
  Compiler produces execution plan (ground truth prediction)
         │
         ▼
  Runtime executes — atom stream records actual outcome
         │
         ▼
  AdaptationCircuit computes delta: plan vs actual
         │
         ▼
  ML model updates: which actions reduced delta? which increased it?
         │
         ▼
  RuntimeDirective: adjusted window sizes, core placement, clock model
         │
         ▼
  Next compilation ingests atom stream as PGO input
         │
         ▼
  Compiler produces improved execution plan
         │
         └──────────────────────────────────────────► loop
```

The loop is tight, deterministic, and self-improving. The compiler's proof is the hard floor — the ML cannot produce an action that violates it. But within the floor, the ML has complete information and complete feedback. Every action has a measured consequence. Every consequence feeds the next prediction. The system does not converge on "good enough". It converges on optimal — optimal for the declared workload, on the declared hardware, within the declared compliance constraints — and it does so continuously, from production data, without human intervention.

This is not a vision of what AI might someday do to operating systems. It is what the clock-aware model makes structurally possible today: a formally verified system that is simultaneously continuously optimised by machine learning, because the model provides the one thing every ML system needs and no conventional system has ever provided — complete, accurate, hardware-sourced ground truth, every tick, forever.

---

## The Logical Conclusion: Intent-Driven Execution

Every layer of the clock-aware model is complete information. The hardware model is declared in `system.cap`. The circuit's intent is declared in its manifest. The execution plan is a compiler-proved theorem. The execution itself is recorded atom by atom in the trace. The feedback loop is closed. Nothing is hidden. Nothing is inferred.

When a system has complete information at every layer, the question changes. The current model of computing asks: *how do I instruct the machine to do what I want?* — and the programmer's job is to translate intent into instructions, manage the resources, handle the errors, and verify the result. The gap between "what I want" and "what the machine does" is the programmer's problem.

In the clock-aware model, that gap can be closed by the machine itself. The programmer declares the outcome requirement — "I need 10 Gbps packet processing at sub-microsecond latency, compliant with DO-178C-Level-A" — and the system derives everything below it:

```
  INTENT DECLARATION
  ─────────────────────────────────────────────────────────
  throughput:    10 Gbps
  latency:       < 1 µs end-to-end
  compliance:    DO-178C-Level-A
  hardware:      system.cap (declared)

         │
         ▼  AI derives circuit structure from intent + hardware model

  CIRCUIT SYNTHESIS
  ─────────────────────────────────────────────────────────
  NicCircuit → channel EthernetFrame → PacketProcessor → ...
  (topology, channel types, handoff tiers — AI-generated)

         │
         ▼  Compiler proves the synthesis meets the intent

  PROOF
  ─────────────────────────────────────────────────────────
  timing proof:     every window fits, every handoff proven
  compliance proof: DO-178C artefact generated
  L1 proof:         zero cache misses on hot path

         │
         ▼  Runtime executes and measures

  EXECUTION + FEEDBACK
  ─────────────────────────────────────────────────────────
  atom stream → delta vs plan → ML update → re-synthesis if needed
```

This is not AI generating code that a human must review. This is AI operating within a formal system — generating circuits that the compiler either proves correct or rejects, executing them under a runtime that measures every consequence, and iterating until the declared intent is met and continuously maintained.

The programmer's role shifts: from instructing the machine in terms it understands to declaring what they need in terms that matter. The machine handles the translation — provably, measurably, without human intervention in the loop.

This is the logical conclusion of making everything declared and everything observable. The gap between intent and execution collapses, because the machine finally has enough information to close it itself.

---

## Simpler Than Anything That Exists

### The Current Stack

Between an application and the silicon, the Linux stack today:

```
C/C++/Rust application
+ libc / stdlib
+ syscall interface
+ VFS / network stack / device drivers
+ kernel memory management
+ scheduler
+ interrupt subsystem
+ hardware abstraction layer
+ firmware / UEFI
+ hardware
```

Each layer: hundreds of thousands of lines. Each interface: documented in 500-page specifications. Each interaction: a potential source of bugs that manifest as behaviour in a different layer. Total: 30+ million lines between the programmer's intent and the silicon executing it.

### The Clock-Aware Stack

```
Application functions
+ Channel stdlib
+ Runtime/kernel (same thing)
+ Assembly stubs (~500 lines)
+ Hardware
```

Five layers. Two of them — runtime and kernel — are the same thing: a collection of declared-timing functions. One of them — the assembly stubs — is ~500 lines of hardware initialisation that has no equivalent in the application model. The application programmer never reads or writes those 500 lines.

The simplification is not cosmetic. The 30 million lines in the Linux stack exist because each layer compensates for information the layer below it could not provide. The scheduler exists because the kernel does not know task duration. The network stack exists because the kernel does not know when the NIC will produce a frame. RCU exists because the kernel does not know when readers finish. Every missing declaration adds a layer.

Declare everything and the layers collapse.

### Why This Matters for Correctness

Complexity is not just an engineering cost — it is a correctness risk. Every interaction between layers is a potential invariant violation. Every abstraction boundary is a place where a correct component can be composed incorrectly with another correct component. The Linux kernel has 30 million lines because the interactions are genuinely complex — and it has tens of thousands of CVEs because the interactions are genuinely complex.

A five-layer stack with two layers being the same thing has fewer interactions by construction. The interactions that remain — channel reads and writes — are typed and subscription-checked at compile time. A correct function cannot be composed incorrectly with another correct function: the channel types enforce the interface, and the subscription list in `system.cap` is the composition specification.

---

## The Complete Comparison

| Property | Today | Clock-Aware |
|---|---|---|
| Languages | Dozens | One |
| Runtimes | Dozens | One |
| OS abstractions | 30M lines | ~500 lines ASM |
| Memory management | GC + malloc + mmap | Declared lifetimes |
| Synchronisation | Locks + RCU + barriers | Declared timing |
| Hardware access | Unsafe escape hatches | Channel types |
| AI assistance | Probabilistic | Compiler-verified |
| Learning curve | Years | Days |
| Certification cost | Millions | Compile time |
| Correctness | Tested | Proven |

---

## Relation to the ADRs

The ADRs in this repository record implementation decisions for a near-term Rust prototype: proc-macro annotations, `system.cap`, `channel T`, `cargo-timeslice`. That prototype is a research vehicle — a way to validate the scheduling model, the channel model, and the verification approach on existing hardware using existing toolchain infrastructure. It is not the OS described in this paper.

The channel-based I/O model (ADR-0010) is the first instantiation of the unified I/O abstraction. The unified system configuration (ADR-0005) is the first instantiation of the hardware-model declaration. The CPU partition model (ADR-0002) is the first instantiation of the static dispatch table. These concepts carry forward directly into the runtime.

Each ADR is a step toward understanding. This paper is the runtime destination.

---

*Part of the clock-aware programming series. See [Paper I: Clock-Aware Programming](01-clock-aware-programming.md) for the core primitive. See [Paper II: The Language](02-language.md) for the language definition. See [Paper IV: Hardware Architecture Implications](04-hardware-architecture.md) for the silicon consequences.*
