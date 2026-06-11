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

### The Compiler Produces a Complete Static Resource Profile

When the compiler compiles a circuit it does not only produce executable code. It produces a complete, static, ahead-of-time picture of everything the circuit will ever use during its lifetime. This profile is part of the manifest — the bitstream the runtime loads.

**Port utilisation timeline.** For every channel the circuit subscribes to, the compiler knows exactly when it reads and when it writes: which tick within which window, at what frequency, producing or consuming what type. The profile is a timeline, not a set of bounds:

```
channel  Channel<RawMessage>   read    tick 0    of every 12-tick window  (consumer)
channel  Channel<Price>        write   tick 10   of every 12-tick window  (producer)
channel  Channel<HandoffLatency> write tick 11   of every 12-tick window  (observability)
```

The runtime loads this profile and hands it to the observability sub-circuit, which uses it as the baseline: if a read or write does not appear at the declared tick, the deviation is itself an observable event. The port utilisation data the observability sub-circuit reports is not sampled — it is the delta between the declared timeline and the measured one.

**Memory footprint by tier.** For every value the circuit declares, the compiler knows its type, its tier, its size in bytes, and its scope:

```
tier     Register    size  8B     scope  expression   count  12 per window
tier     Ephemeral   size  64B    scope  function     count  3 per window
tier     Task        size  128B   scope  window       count  1
tier     Session     size  10MB   scope  circuit      count  1   (RiskLimitTable)
```

The runtime reads this table when slotting the circuit and pre-allocates exactly these amounts from the correct tiers before the first window opens. There is no runtime heap introspection. There is no "how much memory is this process using?" query — the answer was known at compile time and is in the manifest. The runtime's memory accounting is a subtraction: total tier capacity minus the sum of all slotted circuit manifests. Out-of-memory is when that subtraction goes negative, detected at slot time, before any allocation is attempted.

**Physical placement.** The compiler knows — from `system.cap` and the handoff graph — which core each window is assigned to, which cache domain each tier-resident value sits in, and which coherency gap separates each cross-core handoff. This placement is fixed in the manifest. The runtime does not re-derive it. It reads the placement, pins the memory, and programs the dispatch table entry.

The consequence is that the system's complete resource utilisation — across all slotted circuits — is the sum of their manifests. The runtime can answer, at any point, without any measurement: how many ticks on core 2 are allocated; how many bytes of DRAM are committed to `Session` tier; which channels are at peak utilisation; which circuits share a cache domain. These are not runtime metrics gathered by a monitoring daemon. They are compile-time theorems, readable from the manifest table. The observability sub-circuit reports deviations from those theorems. The theorems themselves never change while the system is running.

### Dynamic Memory Is a Channel with a Declared Size

The obvious objection: real applications have dynamic memory. User input is variable length. Network payloads vary. A JSON document arriving over a socket has no fixed size. A compile-time resource profile cannot accommodate data whose size is not known until runtime.

The resolution is already in the model: dynamic data flows through a `Channel<T>`. A channel has a declared size — the ring buffer capacity — fixed at compile time. The data inside the channel is bounded by that size. If the payload is larger than one channel element, it is segmented — multiple elements, same channel, declared capacity. The channel is the declared bound; the content is the runtime value within that bound.

This is not a restriction. It is a clarification of what "dynamic" means. In a conventional system, dynamic memory means: the programmer does not know at compile time how much memory will be needed, so the runtime must provide it on demand from a heap of unknown size. In the clock-aware model, "dynamic" means: the value's content is not known at compile time, but its maximum size is declared — because a circuit that could consume unbounded memory has an undeclared timing contract, and an undeclared timing contract is a compile error.

The practical consequence:

| Conventional dynamic pattern | Clock-aware equivalent |
|---|---|
| `Vec<u8>` growing unboundedly | `Channel<Frame>` with declared `size = 4096` |
| `String` of unknown length | `Channel<Byte>` or segmented `Channel<Chunk>` with declared capacity |
| `HashMap` growing at runtime | `FlatMap<K,V>` with declared `size` — maximum key count declared |
| Dynamic JSON parse tree | `Channel<JsonToken>` — streaming tokeniser, fixed buffer, no tree allocation |
| Recursive data structure | Iterative traversal over declared-size collection — recursion depth is timing; declare it |

A streaming JSON parser is the clearest example. A conventional parser builds a tree on the heap — unbounded allocation, unknown depth, unknown lifetime. A clock-aware parser reads tokens from `Channel<RawByte>` and writes structured events to `Channel<JsonToken>` — one token at a time, fixed buffer sizes, no allocation, no tree. The application receives a stream of typed events and processes each one in its declared window. The document may be arbitrarily large; the memory footprint is constant and declared.

For genuinely unbounded data — a file upload with no declared maximum, a stream that never terminates — the channel's declared `size` is the processing window. The circuit processes `size` elements per window and requests more in the next window. The throughput is bounded by the declared window frequency, not by memory. The circuit's memory footprint remains constant regardless of total data volume.

The model does not eliminate dynamic data. It eliminates dynamic memory — the condition where memory size is unknown at compile time. Every circuit processes within a declared footprint. The data flowing through it may be arbitrarily variable; the container it flows through is not.

### Where the Model Requires Honest Discipline: SQL and Full Materialisation

SQL is the hardest case. A query `SELECT * FROM orders WHERE date > '2024-01-01'` may return 10 rows or 10 million. The result set size is not known at compile time and cannot be declared without knowing the data. This is a genuine tension.

The model resolves it cleanly for the majority of SQL operations — and honestly does not for one specific case.

**What works without compromise:**

Streaming queries. The database circuit produces `Channel<Row>` with a declared buffer size — say, 1024 rows. The consumer processes each batch in its declared window and reads the next batch in the following window. The total result set may be arbitrarily large; the memory footprint is `1024 × sizeof(Row)`, declared at compile time. The frontend streams the same channel — rendering rows as they arrive, never holding the full result in memory. Aggregations — `COUNT`, `SUM`, `AVG`, `MIN`, `MAX` — run in a single streaming pass. No materialisation. Declared footprint.

**What requires a declared maximum:**

`ORDER BY` on an unbounded result set requires materialising the full set before sorting — unless the data is pre-sorted at write time, or unless the sort is pushed into the database circuit and applied at ingestion. If the application must sort at query time on an arbitrarily large result, it must declare a maximum: `FlatMap<RowKey, Row>` with `size = 100_000`. If the result exceeds that maximum, the circuit emits a `Channel<TruncationEvent>` — a declared signal, handled by the exhaustive match, not a silent overflow. The programmer must decide: declare a maximum and acknowledge truncation, or re-architect the query to push the sort upstream.

This is not a failure of the model. It is the model being honest about what the programme is doing. A sort on an unbounded result set has undeclared memory — the model requires it to be declared. The conventional alternative is a heap that silently grows until the process is killed by the OOM killer. The clock-aware alternative is a declared maximum and an explicit truncation signal. The constraint is real; so is the conventional alternative's behaviour.

**The architectural push:**

The model pushes SQL-style applications toward streaming-first design: declare the maximum result set at the query boundary, push aggregation and ordering into the storage layer where the data is already sorted by write pattern, render the frontend incrementally. This is not a new pattern — it is what high-throughput data systems already do. The clock-aware model makes it the only pattern, rather than the recommended one.

### The Weak Point Dissolves: Every Production System Already Declares Bounds

The apparent weakness of the declared-footprint model — that `Session` and `Permanent` tier sizes must be declared at compile time — dissolves on inspection. No production system runs with truly unbounded memory. Every deployed JVM has `-Xmx`. Every production PostgreSQL has `shared_buffers`. Every Kubernetes pod has a memory limit. Every embedded system has a linker script with fixed section sizes. The bounds are always declared somewhere — in a config file, a deployment manifest, a command-line flag, an operations runbook.

The difference is where the declaration lives and when it is verified. In a Java application, the heap bound is a runtime flag passed to the JVM at launch — not verified against the programme's actual memory behaviour, not checked against the data structures the programme allocates, not proved consistent with the programme's timing. The flag is an operational constraint applied outside the language. The programme can be written to require more than the flag permits, and nobody will discover this until the JVM throws `OutOfMemoryError` in production.

In the clock-aware model, the bound is declared in the programme itself — as a `Session` or `Permanent` type with a declared size — and the compiler verifies it is consistent with everything the circuit accesses. The declaration is the same fact the operations team would have put in `-Xmx`. It is just in the right place: in the source, verified at compile time, part of the manifest the runtime loads.

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

**Ongoing arbitrage — adaptive shedding.** The observability sub-circuit continuously compares actual port utilisation against the declared profile. When a circuit consistently underutilises its declared channel bandwidth or finishes well within its budget, the runtime can signal it via `Channel<ResourceAdvisory>` — a declared channel the circuit may subscribe to — suggesting it yield surplus tick windows back to the pool. The circuit's exhaustive match handles the advisory: it may reduce its declared footprint, defer non-critical work to `@Cold` windows, or ignore the advisory if the architect declared that circuit weight-exempt. The decision is always in the circuit's match, not in the runtime's policy.

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

This is identical to an application function. The only distinction between a kernel function and an application function is the channel subscription list. Kernel functions subscribe to hardware-backed channels (`Channel<IrqSignal>`, `Channel<TimerTick>`). Application functions subscribe to logical channels (`Channel<NicFrame>`, `Channel<OrderDecision>`). In both cases the programmer writes the same channel API — the compiler resolves the physical backing from `system.cap`. Both are compiled with the same proc-macro, verified against the same `system.cap`, sealed into the same signed manifest.

The runtime is not a separate thing that sits between the language and the kernel. The kernel is the runtime. Both are collections of declared-timing circuits, compiled and verified by the same toolchain.

### The OS Is a Collection of Circuits

The operating system is not a special-privileged process. It is a collection of declared-timing circuits, written in the language, compiled by the same toolchain, verified by the same compiler. Apart from the ~500 lines of Assembly stubs that handle boot and hardware initialisation, every line of the OS is a function or a class with a `@Timeslice` annotation, a lifetime type, and a channel subscription list.

There is no privileged mode at the language level. The distinction between an OS circuit and an application circuit is the channel subscription list: OS circuits subscribe to hardware-backed channels (`Channel<IrqSignal>`, `Channel<TimerTick>`, `Channel<DmaCompletion>`); application circuits subscribe to logical channels (`Channel<NicFrame>`, `Channel<OrderDecision>`). The language is the same. The compiler is the same. The four rules are the same.

### Handoff: Declared Cycle Boundaries, Not Locks

OS circuits communicate by writing to declared registers or memory locations at declared cycle boundaries. Circuit A writes at cycle N. Circuit B reads at cycle N+1. The compiler accounts for the timing of the handoff: it knows A's write cycle and B's read cycle, and proves that B's read is scheduled strictly after A's write. No synchronisation primitive is needed — the timing proof is the synchronisation.

This is not shared memory protected by a lock. Shared memory protected by a lock is a compensating mechanism for not knowing when two threads will access the memory. Declare when they access it and the lock is structurally unnecessary. The memory location is not "shared" in any meaningful sense — it is a typed channel between two declared-timing circuits, backed by a register or DRAM location chosen by the compiler from the declared lifetime tier.

### Live Observability: Dedicated Sub-Circuits

The OS contains a layer of dedicated sub-circuits whose sole function is observability. These circuits run in their own declared cycle windows, isolated from hot-path application circuits:

- **Port utilisation** — per-channel read/write counts, saturation, backpressure events
- **Subsystem registry** — which circuits are present in the current compiled configuration
- **Active/idle status** — per-circuit: currently executing, waiting at cycle boundary, or idle (not configured)

These sub-circuits do not poll. They produce `Channel<SystemSnapshot>` at a declared low-frequency cycle (e.g. every 100 ms). The observability data is always fresh — not sampled, not aggregated by a runtime monitor that wakes up occasionally — because the sub-circuits are always running in their declared windows. A circuit that fails to reach its cycle boundary produces a gap in the snapshot stream, which is itself an observable event.

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

The runtime supplies a default error-handler for every signal type. The default handlers are conservative: log the signal to the observability channel, remove the circuit cleanly, notify any declared dependents via `Channel<DependencyLost>`. A programmer who does not declare a custom handler gets this behaviour automatically — and the compiler records in the manifest which signals are handled by the default and which have an explicit override. A programmer who declares a custom error-handler circuit replaces the default for the specific signal types it covers; the remainder fall through to the default. There is no way to leave a signal type unhandled: either the programmer's handler covers it, or the default does.

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
                          │        │ Channel<ErrorSignal> │
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
| `ClockCircuit` | Reads the hardware cycle counter (TSC). Calibrates it to wall-clock time. Does not drive the scheduler — the CPU clock does that directly. | `Channel<Tick>`, `Channel<WallClock>` |
| `MemoryCircuit` | Pre-allocates memory regions at slot time. Manages tier promotion/demotion. Handles circuit removal and reclaim. | `Channel<SlotAck>`, `Channel<RemovalSignal>` |
| `NicCircuit` | DMA-paced network I/O via the NIC driver. Delivers frames to subscriber circuits at declared cycles. | `Channel<EthernetFrame>` |
| `NamespaceCircuit` | Maps names and identifiers to `Cold` tier regions. The filesystem namespace. | `Channel<ColdRegion>` |
| `ObservabilityCircuit` | Publishes per-circuit utilisation, timing deltas, and system snapshots at a declared low-frequency window. | `Channel<SystemSnapshot>` |
| `TerminalCircuit` | Reads keyboard input. Resolves names in the namespace. Calls `add_circuit`. Displays output. | `Channel<DisplayOutput>` |

`ClockCircuit` is worth dwelling on. In a conventional OS, the timer interrupt is what drives the scheduler: it fires periodically, the kernel preempts the running process, picks the next one. In the clock-aware model, the scheduler *is* the CPU clock — the dispatch table encodes who runs at which tick, and the clock advances it. `ClockCircuit` has nothing to do with scheduling. It is purely a source of time-as-a-value: circuits that need elapsed time, timeouts, or wall-clock timestamps subscribe to its channels. The scheduling mechanism and the timekeeping mechanism are completely separate, which is the correct factoring.

### Callable and Non-Callable Circuits

Not all circuits are equal from the user's perspective. The OS circuit collection includes a `TerminalCircuit` — a circuit that subscribes to `Channel<KeyboardInput>`, parses commands, and dispatches to other circuits by name. It is the user-facing surface of the system. What it can dispatch to defines the distinction between circuit kinds:

**Callable circuits** are registered in the `NamespaceCircuit` with a public name. The terminal can look them up by name and invoke them via `add_circuit`. They produce output to a `Channel<DisplayOutput>` that the terminal subscribes to for the duration of their execution. They are the clock-aware equivalent of POSIX commands — `ls`, `cp`, `grep`, and any user-defined program that declares itself callable. A callable circuit is a first-class citizen of the namespace: it has a name, a manifest, a declared signature, and can be composed with other callable circuits via channel piping.

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

**EventDrivenDriver** — subscribes to a hardware interrupt or DREQ assertion. Fires when the hardware signals readiness. Produces to a `Channel<T>` that application circuits can subscribe to.

**MessageDrivenDriver** — transfers data through channels. Three orientations:

| Orientation | Description |
|---|---|
| `ReadOnly` | Consumes from `Channel<T>`. Produces nothing. Used for write-only hardware: DAC, actuator, display. |
| `WriteOnly` | Produces to `Channel<T>`. Consumes nothing. Used for read-only hardware: ADC, sensor, NIC RX. |
| `BothWays` | Consumes from one channel, produces to another. Full-duplex hardware: SPI, UART, Ethernet with RX+TX. |

A driver declares its channels, its timing constraints, and its DMA configuration. The compiler resolves the DMA ring buffer, the DREQ line, and the interrupt vector from the device descriptor in `system.cap`. The driver programmer writes `Channel<UartFrame>` — the compiler wires the UART DREQ to the DMA controller and delivers completed frames to the channel at the declared cycle. No manual DMA ring setup. No interrupt registration. No `request_irq`. No `dma_alloc_coherent`.

### A Device Is a Class — A Stateful Circuit with Declared Signal Access

A device is more precisely a `class` — a stateful circuit whose register file is the device's internal state, and whose input and output ports are accessible to the kernel via declared signals. Three device kinds:

```
class ReadOnlyDevice<T> {
    // Device produces data. Kernel reads.
    // Input: hardware DREQ signal — is data ready?
    // Output: Channel<T> — the data, delivered when DREQ asserts
    val ready: Signal   // DREQ line — compiler monitors this
    val out:   Channel<T>
}

class WriteOnlyDevice<T> {
    // Kernel sends data. Device consumes.
    // Input: Channel<T> — data to send
    // Output: hardware accept signal — is the device ready to receive?
    val accept: Signal  // device's write-ready line
    val in:     Channel<T>
}

class ReadWriteDevice<In, Out> {
    // Full-duplex. Both directions, independent signals.
    val ready:  Signal       // DREQ: device has data for kernel
    val accept: Signal       // device ready to receive from kernel
    val in:     Channel<In>
    val out:    Channel<Out>
}
```

The `Signal` is the DREQ line — a single hardware bit that answers the question "is the device ready?" for its direction. The kernel does not poll it. The compiler subscribes the driver circuit to the signal, and the DMA controller triggers a transfer the moment the signal asserts. The circuit's next window opens with the data already in the channel — no wait, no syscall, no ISR to schedule.

```
  ReadOnlyDevice (e.g. NIC RX)

  hardware ──► DREQ asserts ──► DMA transfers ──► Channel<EthernetFrame> written
                    │
                    └── compiler-wired: driver circuit fires on next clock edge
                                        after DREQ assertion, not before

  WriteOnlyDevice (e.g. DAC, actuator)

  Channel<Sample> written ──► accept asserts? ──► DMA transfers ──► hardware
                                    │
                                    └── compiler holds transfer until accept is high:
                                        never writes to a device that isn't ready
```

This completes the connection. The device declares its readiness via `Signal`. The compiler wires the signal to the DMA trigger. The DMA delivers data to or from the channel. The downstream circuit's window opens with the data present. Everything between DREQ assertion and circuit execution is compiler-resolved — no runtime discovery, no interrupt handler to schedule, no quiescent state to emerge from. The signal is up: data flows. The signal is down: nothing moves. The hardware tells the truth; the compiler listens at build time.

### Everything Is DMA. Everything Is Subscription-Based.

The driver never polls. The driver never blocks. There is no path from hardware I/O to CPU that goes through a polling loop or a blocking syscall. Every hardware data transfer in the OS is a DMA operation, scheduled by the compiler, paced by the hardware's own DREQ signal.

Application circuits cannot read hardware channels directly. They subscribe through the driver that owns the channel. The driver declares which `Channel<T>` it exposes and which subscriber identities are permitted, in its subscription list in `system.cap`:

```
driver.Uart0.exposes = Channel<UartFrame>
driver.Uart0.permits = [SerialLogger, DebugMonitor]
```

This is the permission model. Not a syscall table. Not a capability token. Not a privilege ring. A channel subscription, declared in `system.cap`, verified by the compiler at build time. A circuit that attempts to read a channel not listed in its subscription declaration is a compile error. The permission check is not a runtime gate — it is a proof obligation discharged at compile time.

---

## Execution

### The Runtime Is the Scheduler

The scheduler exists to resolve contention between tasks whose timing is unknown. In a clock-aware system, there is no such contention — every function's window is declared non-overlapping. There is nothing to arbitrate. The "scheduler" in this model is the clock itself: functions execute at their declared cycles, in order, because the CPU executes the next instruction. No arbitration. No priority queue. No run queue. No preemption. The cycle annotation is the schedule; the hardware is the scheduler.

### How Cycles Work: What the Compiler Solves

This is the mechanism. The question "how does the system actually run by cycles" has a precise answer at every level.

**Step 0 — Read the hardware.**

Before translating a single declaration, the compiler needs two numbers from the hardware: clock frequency and pipeline depth. Both are available without guessing.

On x86, `CPUID` leaf `0x15` returns the TSC frequency directly in Hz. On ARM, the system register `CNTFRQ_EL0` holds the counter frequency. On either architecture, the BIOS/UEFI exposes the same figure in the ACPI `FADT` table. The compiler reads it at build time — not at boot, not at runtime. The frequency is a fact about the physical chip, declared in silicon, readable before the first instruction of the programme executes.

Pipeline depth is equally concrete. A modern CPU pipeline is a fixed number of stages: Skylake is 14–19 stages, Cortex-A76 is 13. This is published in the Intel/ARM optimisation manuals and also exposed in `cpuid`'s model-specific data. Pipeline depth tells the compiler two things: the minimum instruction latency for a dependent chain (you cannot issue the next instruction until the pipeline has resolved the previous one's result), and the penalty for a branch misprediction (pipeline flush = N stages × 1 tick). Both affect worst-case instruction count. Both are known statically for a declared `cpu_model`.

The programmer does not read any of this. It goes into `system.cap` once, as `cpu_model = "intel-skylake-3ghz"`, and the compiler's model for that CPU supplies frequency, pipeline depth, execution unit counts, store-forward latency, and cache-line size — the full machine model. If `cpu_model` is not in `system.cap`, the compiler refuses to produce a binary. Timing correctness cannot be proved without knowing the machine.

**Step 1 — Translate declarations to ticks.**

`@Timeslice(core = 2, cycle = "4ns", budget = "3.5ns")` says: on core 2, this circuit has a 4 ns window and must finish within 3.5 ns. The compiler reads `cpu_model` from `system.cap` — say, 3 GHz — and translates: one cycle = 0.33 ns, so 4 ns = 12 ticks, budget = 10.5 ticks. All timing from this point is in integer tick counts, not nanoseconds. The hardware knows nothing about nanoseconds.

**Step 2 — Count the instruction cycles.**

The compiler feeds the compiled circuit body to `llvm-mca` (or its equivalent in the self-hosted toolchain). `llvm-mca` models the CPU's execution units, pipeline depth, and memory latencies for the declared `cpu_model` and returns the worst-case instruction count. Instruction ordering matters here: the compiler's backend arranges the instruction sequence to avoid pipeline stalls — an instruction whose input depends on the output of the instruction N stages ahead must be separated by at least N ticks, or a stall fills those ticks with nothing. The compiler knows N because it knows the pipeline depth from Step 0. It reorders, fills gaps with independent instructions, and only fails with a compile error if the worst-case count — after optimal ordering — still exceeds the budget.

**Step 3 — Build the static dispatch table.**

The compiler collects every circuit in the system — OS circuits and application circuits — and solves for a static, non-overlapping assignment of tick windows to cores:

```
core 0, tick   0: ClockCircuit      (window: 6 ticks)
core 0, tick   6: MemoryCircuit     (window: 8 ticks)
core 2, tick   0: parsePrice        (window: 12 ticks)
core 2, tick  12: updateBook        (window: 10 ticks)
core 3, tick   0: ObservabilityCircuit (window: 300M ticks)
```

This is a constraint satisfaction problem: assign windows such that (a) no two circuits share a core at the same tick, (b) every window ≥ its `llvm-mca` worst-case count, (c) every handoff pair satisfies writer_window_end < reader_window_start. If the constraint system is unsatisfiable — the declared circuits do not fit on the declared cores within their declared budgets — it is a compile error, naming the conflicting circuits and the shortfall in ticks. The system does not ship with a timing conflict hidden inside it.

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

When multiple applications are already running, their interactions are exclusively through declared memory handoffs — values written at a declared tick boundary by one circuit and read at a declared tick boundary by another. The compiler already proved those handoffs valid for every existing circuit pair. Adding a new circuit that connects to an existing circuit via a declared `Channel<T>` extends the handoff graph by one edge. That edge is verified at compile time against the live timing: the new circuit's read window must open after the existing circuit's write window closes, accounting for the live phase offset. If it does, the handoff is valid by the same theorem as every other handoff in the system. There is no runtime arbitration — the memory location is not contested. One circuit writes at tick N. One circuit reads at tick N+k. The compiler proved k ≥ coherency latency. The hardware clock enforces it. The handoff is the arbitration.

### The Performance Mechanism: Pipelining

Cycles executing in order is not a constraint — it is the source of performance. The dispatch table is a software pipeline. Each circuit is a stage. The clock advances the pipeline. The compiler's job is to arrange the table so that data is always exactly where the next stage needs it, at the tick the next stage starts.

The mechanism depends on the declared lifetime tier of the value being handed off:

**Register-based handoff — same core, consecutive windows.**

If the value's lifetime is `Register` or `Ephemeral`, it never needs to leave the CPU's register file. The compiler arranges consecutive windows on the same core so that the output registers of window N are the input registers of window N+1. No store. No load. No memory traffic at all. The handoff costs zero cycles because the data is already in the right place. This is identical to how a hardware pipeline passes a value between stages through a flip-flop: the register is the pipeline register.

**Memory-based handoff — same core, higher lifetime.**

If the value's lifetime is `Task` or `Session`, it lives in L1 or L2 (pinned). The compiler emits the store at the last instruction of window N. The reader's window N+1 opens on the same core immediately after. The store-to-load forwarding path in the CPU delivers the value from the store buffer before it even reaches L1 — the latency is zero observable cycles for the reader. The compiler knows the store-forward latency for the declared `cpu_model` and schedules the window gap to be ≥ that latency. The latency is absorbed in the gap between windows, charged to neither circuit's budget.

**Cross-core handoff — different cores, same or different dies.**

The compiler reads the cache topology from `system.cap`: same L3 domain, different L3 domain, or NUMA. Each topology has a known coherency latency in ticks. The compiler adds a mandatory gap between the writer's window close and the reader's window open on the other core, equal to the coherency latency for that topology. The reader's window does not open until the data is guaranteed coherent. The handoff latency is a theorem in the dispatch table — not a runtime measurement, not a best-effort estimate, a verified tick count.

**The system as a whole is a deeply pipelined machine.**

```
core 2, tick  0–12:  parsePrice     → produces Channel<Price>   (Register lifetime)
core 2, tick 12–22:  updateBook     → consumes Channel<Price>   (same core, register handoff, 0 cycles)
core 2, tick 22–34:  emitQuote      → consumes OrderBook state  (Task lifetime, L1 pinned)
core 0, tick 14–30:  riskCheck      → consumes Channel<Price>   (cross-core, gap ≥ coherency latency)
```

The compiler proves this pipeline is valid: every stage receives its input before its window opens, every stage finishes before its window closes, every handoff latency is absorbed in a compiler-computed gap. No pipeline stall is possible — because a stall would mean a circuit exceeded its budget, which is a compile error.

This is what Vivado does with FPGA timing constraints: it solves for register-to-register paths, verifies each path meets the clock period, and reports setup/hold violations at synthesis time. The clock-aware compiler does the same thing in software, for a CPU, with a static dispatch table instead of a place-and-route netlist.

The CPU's own hardware pipeline — out-of-order execution, store forwarding, prefetcher — runs inside each window, invisible to the compiler. The compiler's pipeline runs across windows. Two levels of pipelining, both proven, both counted.

### The Trace Unit — Proving the Compiler Was Right

On ARM (and equivalent architectures), the trace unit timestamps every instruction as it commits. The clock-aware runtime subscribes to the trace unit's output as a `Channel<TraceEvent>` through the `ObservabilityCircuit`. This gives the system something no conventional OS has ever had: a continuous, hardware-sourced, instruction-level audit of every circuit's actual execution.

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

The compiler emits an **execution plan** alongside the binary — the expected sequence of instruction timestamps, register values, and port utilisations for the hot path. The `ObservabilityCircuit` continuously compares the trace stream against the execution plan. A deviation is a `Channel<ExecutionPlanViolation>` signal: the specific instruction, the expected cycle, the actual cycle, and the delta.

This produces three guarantees that no conventional system can make:

1. **Zero cache misses, proven in real time.** Every cache miss on the hot path is a trace event. If the execution plan predicts zero misses and the trace shows one, a violation signal fires immediately — not in a post-mortem profiler, not in a benchmark report, in the same execution window.

2. **Misprediction confirmation.** There are no branches on the hot path by construction. If the trace unit reports a branch prediction event in a circuit whose hot path the compiler eliminated all branches from, it confirms the compiler model is correct — or flags a hardware anomaly in the CPU itself.

3. **Execution plan validation.** A circuit that consistently executes faster than its declared budget can donate tick windows back to the runtime pool. A circuit that consistently approaches its budget boundary signals the `ObservabilityCircuit` before it becomes a violation. The runtime reacts through declared channels, not through forced intervention.

On ARM specifically, the ETM (Embedded Trace Macrocell) provides this at silicon level with sub-cycle resolution. The clock-aware runtime treats it as a first-class subscription source — not a debug facility to be attached occasionally, but a permanently running circuit producing a continuous stream of hardware-sourced proofs.

The trace unit is also the input to PGO. Real execution traces from production replace synthetic profiles. The next compilation ingests the trace, adjusts instruction ordering and register allocation to match actual hotspots, and produces a binary that is provably better than the previous one — not heuristically better. The trace is a theorem input.

---

## Memory

### The Runtime Is the Memory Manager

Memory management is determined by lifetime types, which are determined by cycle annotations. The compiler knows at build time which memory tier every value lives in and when it expires. At the cycle boundary, memory is reclaimed automatically, without a single runtime instruction dedicated to collection.

The two-question memory manager:
1. Did the function request allocation this cycle? → serve from the declared tier.
2. Did the function finish this cycle? → reclaim its entire working set.

Every GC algorithm ever invented — generational, concurrent, incremental, ZGC, Shenandoah — is a runtime approximation of this two-question circuit. The approximation exists because the GC does not know the mutator's timing. Declare the timing and the approximation becomes unnecessary.

### The Runtime Is the Sole Memory Provider

No circuit allocates memory. No circuit calls `malloc`. No circuit holds a pointer to an allocator. The runtime is the only entity in the system that provides memory — and it does so exclusively by serving declared lifetime types from the correct tier.

When a circuit is slotted into the dispatch table, the runtime reads its manifest: every value it declares, its tier, its size, its scope. The runtime pre-allocates the circuit's entire working set from the appropriate tier before the circuit's first window opens. The circuit's first instruction executes into memory that is already present, already pinned, already at the correct cache level. There is no allocation on the hot path — ever. The allocation happened at slot time, not at execution time.

This has a precise consequence for reclaim. Because the runtime provided every byte the circuit uses, and because every byte's lifetime is declared in the manifest, the runtime knows exactly when each value expires:

| Lifetime | Reclaim trigger | Reclaim cost |
|---|---|---|
| `Register` | Expression boundary | Zero — register file reuse |
| `Ephemeral` | Function return | Zero — L1 slot reuse |
| `Task` | Cycle window close | Zero — L1/L2 slot reuse |
| `Session` | Circuit removal | Single range deallocation |
| `Permanent` | Programme termination | Single range deallocation |
| `Cold` | Declared unload cycle | NVMe page reclaim |

"Reclaim cost" is zero for the hot tiers because there is nothing to collect — the slot was pre-allocated, the window closed, the same slot is used for the next circuit's matching tier. There is no free list to walk, no reference count to decrement, no mark phase, no sweep phase. The runtime increments a pointer. The slot is reclaimed.

For `Session` values — cross-circuit handoffs like `riskLimits` — reclaim happens when the owning circuit is removed from the dispatch table. The runtime reads the circuit's manifest, walks its declared `Session` values, and releases each range back to the DRAM pool in a single operation. The cost is proportional to the number of declared `Session` values in the manifest, not to the size of the data. The runtime never touches the data itself — it reclaims the address range. A `Session<RiskLimitTable>` with 10 MB of data costs the same to reclaim as one with 10 bytes.

This is why the runtime must be the sole provider. A circuit that allocated its own memory would hold a range the runtime does not know about. The runtime could not reclaim it at circuit removal without scanning — which requires knowing where to scan, which requires the circuit to have declared it. The declaration is the allocation. The allocation is the declaration. The runtime holds both ends of the same fact.

The programmer never sees this. They declare `Session<RiskLimitTable>` and write to it. The runtime provided the memory before the first write. The runtime reclaims it after the last read. The circuit never held a pointer to an allocator, never called a destructor, never decremented a reference count. The lifecycle is not managed — it is declared. The runtime executes the declaration.

### Out of Memory Is a Circuit Removal Event

When a new circuit is slotted and the runtime attempts to pre-allocate its declared working set, it may find the declared tier is full. In a conventional OS this is an out-of-memory condition — a crisis that triggers the OOM killer, which scans process tables, scores processes heuristically, kills one, hopes the kill frees enough memory, and logs the outcome after the fact.

In the clock-aware model there is no crisis. The runtime knows, before allocating a single byte, exactly which circuits currently occupy the full tier and exactly how much memory each one holds — because the runtime provided every byte to every circuit and holds the manifest record of each allocation. Out of memory is not a scan problem. It is an arithmetic problem: the new circuit needs N bytes; the tier has M bytes free; M < N; remove a circuit to recover at least N − M bytes.

The removal protocol:

1. **The runtime signals the circuit selected for removal.** The signal is delivered as a `Channel<RemovalSignal>` write — the same mechanism as any other inter-circuit communication. The selected circuit receives the signal at its next declared window open.

2. **The circuit acknowledges and completes its current window.** It does not abort mid-execution. Its current window runs to completion — the timing guarantee is not violated by removal. At the window close, the circuit writes a `Channel<RemovalAck>` back to the runtime.

3. **The runtime removes the circuit from the dispatch table.** Its windows are released. Its channel subscriptions are unregistered. Its entire declared working set — every `Session`, every `Task`, every `Permanent` value in its manifest — is reclaimed in a single manifest-walk. No scan. No pointer traversal. The runtime reads the manifest entries and releases the address ranges.

4. **The cause is written to the observability log.** The removal event is a `Channel<CircuitEvent>` write to the observability sub-circuit, containing: circuit identity, removal cause (`OutOfMemory`), tier that was exhausted, bytes reclaimed, timestamp in ticks, and the identity of the circuit whose slot request triggered the removal. The log entry is structural — it is a typed channel write, not a string printed to stderr.

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

**Caching** is the tier system itself. Data that starts in the `Cold` tier (NVMe, demand-loaded) and is promoted to `Permanent` or `Session` (DRAM, pinned) *is* the cache — with the promotion policy declared at compile time rather than managed by a page replacement algorithm at runtime. There is no separate buffer cache to maintain. There is no `drop_caches`. The tier system is the cache, and the runtime is its manager.

**The namespace** — the mapping from names to locations in the `Cold` tier — is the one part that requires a dedicated circuit. This is the `NamespaceCircuit`:

```
  name / identifier
         │
         ▼
  ┌─────────────────────┐
  │   NamespaceCircuit  │
  │                     │
  │  name ──► Cold<T>   │   reads/writes via NVMe driver circuit
  │           region    │──────────────────────────────────────►
  │           + offset  │
  │           + size    │
  └─────────────────────┘
         │
         ▼
  Channel<ColdRegion>   (returned to caller at declared cycle)
```

The `NamespaceCircuit` maps identifiers — circuit names, handoff keys, manifest hashes, application-level paths — to declared `Cold` regions. It holds no cache of its own; the tier system already promotes hot regions into DRAM. It performs no journaling in the traditional sense; durability is the declared write-commit policy of the `Cold` tier, enforced at the clock boundary by the NVMe driver circuit. It holds no locks; lookups are channel reads, writes are channel writes, and conflicting writes to the same region are resolved at the clock boundary by the runtime's ownership model.

| Traditional filesystem concern | Clock-aware equivalent | Where it lives |
|---|---|---|
| Block I/O | NVMe driver circuit | OS circuit collection |
| Page cache / buffer cache | `Permanent` / `Session` tier promotion | Runtime tier manager |
| Cache eviction policy | Tier demotion on memory pressure | Runtime (declared weights) |
| Journal / durability | `Cold` write-commit at clock boundary | NVMe driver circuit |
| Namespace (paths, inodes) | `NamespaceCircuit` | OS circuit collection |
| File locking | Channel ownership at clock boundary | Runtime ownership model |
| `open` / `close` / `mmap` | `Cold<T>` channel subscription | Language type system |

There is no `VFS` layer. There is no `inode` table separate from the namespace. There is no `dentry` cache separate from the `Permanent` tier. There is no `fsync` — the write-commit cycle boundary *is* the sync. A programme that holds a `Cold<T>` handle is subscribed to that region; when it drops the handle (removes the subscription), the runtime releases the region back to the `NamespaceCircuit`. There is no file descriptor table. There is no `open file description`. There is a typed channel subscription — the same primitive used for every other resource in the system.

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

### The Runtime Adapts — AI-Regulated OS

The atom stream, the ML execution planner, and the clock model assignment together form a system that adapts in real time to the actual workload — not by guessing, not by sampling, but by reading a hardware-sourced proof stream and acting on it within the constraints of the compiler's theorems.

The logical conclusion of this architecture is that the OS itself can be regulated by an AI circuit: a declared `AdaptationCircuit` that subscribes to `Channel<AtomStream>`, runs inference over the execution history, and emits `Channel<RuntimeDirective>` — window size adjustments, core placement recommendations, clock model pre-assignments — that the runtime applies at the next dispatch boundary.

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
| Out of memory | `Channel<RemovalSignal>` | Runtime → circuit |
| Dependency circuit removed | `Channel<DependencyLost>` | Runtime → subscriber circuits |
| Budget overrun detected | `Channel<BudgetViolation>` | Observability sub-circuit → declared handler |
| Hardware fault | `Channel<HardwareFault>` | Driver circuit → subscriber circuits |
| Handoff staleness exceeded | `Channel<StalenessViolation>` | Runtime → declared handler |

Each signal is a typed channel write. The receiving circuit handles it in its next window via its declared exhaustive match — the same mechanism it uses for any other input. There is no separate error-handling path. There is no control-flow transfer outside the circuit's declared windows. The circuit reads the signal, matches it, executes the declared handler, and continues — or acknowledges removal and exits cleanly.

The consequence is that error handling is visible, declared, and compiler-verified in exactly the same way as the happy path. A circuit that subscribes to `Channel<RemovalSignal>` but does not handle it in its match is a compile error. A circuit that handles `DependencyLost` with `ignore` has explicitly declared that it tolerates losing its dependency — and the compiler records that declaration in the manifest. There are no silent failure modes. There are no swallowed exceptions. There are no unhandled panics that terminate the process and leave the system in an unknown state.

A circuit that exceeds its declared budget is not killed. The budget overrun is written to the observability channel. The declared `BudgetViolation` handler runs — which may log the event, reduce the circuit's load, or signal a downstream circuit to shed work. The system adapts through declared channels, not through forced termination.

The signal model unifies what other systems treat as three separate concerns — exceptions, signals, and inter-process communication — into one: a typed channel write, handled by an exhaustive match, at a declared cycle. One mechanism. No exceptions.

An exception is an imperative pattern. It is the runtime seizing control from the programme because the programme did not declare what to do. It belongs to the same family as the scheduler, the GC, the OOM killer, and the lock: compensating machinery that exists because something was never declared. Declare it and the machinery is unnecessary. The clock-aware model does not improve exception handling — it makes the condition that requires it structurally impossible to create.

---

## Services and Security

### Microservices Are Circuits with Declared Handoffs

This model scales directly to what is conventionally called a microservice architecture — except without the network, without the serialisation, without the service mesh, without the latency. A microservice is a circuit. Its interface is its declared `Channel<T>` subscriptions. Its deployment is adding it to the live system. Its communication with other services is a memory handoff at a declared tick boundary.

The handoff tier determines what kind of inter-service communication you get:

| Handoff tier | Physical backing | Latency | Use |
|---|---|---|---|
| `Register` | CPU register file | 0 ticks | Same-core consecutive circuits only |
| `Ephemeral` | L1 cache, pinned | 1–4 ticks | Same-core, tight pipeline |
| `Task` | L1/L2, pinned | 4–12 ticks | Same-core, wider window |
| `Session` | DRAM, resident | 50–200 ticks | Cross-core, long-lived shared state |
| `Permanent` | DRAM, pinned, never paged | 50–200 ticks | Cross-circuit shared tables, reference data |
| `Cold` | NVMe, demand-loaded | declared cycle | Infrequent bulk data |

A trading engine and a risk checker on adjacent cores hand off a `Session<RiskLimit>` table: the engine writes updated limits at the end of its window; the risk checker reads them at the start of its window on the next core. No HTTP. No gRPC. No message broker. No serialisation. A memory location, a declared write tick, a declared read tick, and a compiler proof that the read comes after the write.

The programmer can annotate handoff points to give the compiler and runtime additional information:

```java
@Handoff(tier = Session, from = "TradingEngine", to = "RiskChecker", maxStaleness = "1ms")
val riskLimits: Session<RiskLimitTable>
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
  │             │          Session<RiskLimitTable>        ││                   │
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
| Memory tier | `tier = Session` | Pins `riskLimits` in DRAM for programme run lifetime. Verifies declared size fits tier. |
| Staleness bound | `maxStaleness = "1ms"` | Translates to tick count (e.g. 3,000,000 ticks at 3 GHz). Verifies `TradingEngine`'s write period ≤ that count. Compile error if not. |
| Coherency gap | `system.cap` cache topology | Reads L3 domain membership for core 1 and core 2. Inserts mandatory gap = cross-L3 coherency latency (e.g. 60–80 ticks) between write window end and read window start. |
| Core placement | producer + consumer pair | Attempts to place `TradingEngine` and `RiskChecker` on cores sharing an L3 domain. If unavailable, uses next-best topology and recalculates gap. |
| Staleness logging | `maxStaleness` declared | Emits a `RDTSC` delta instruction at `RiskChecker`'s `riskLimits.get()` call site. Writes delta to `Channel<HandoffLatency>` for the observability sub-circuit. |
| Anonymity prevention | both sides named | A value annotated `@Handoff` with no matching write in `from` or no matching read in `to` is a compile error. The memory location cannot be unowned. |

`@Handoff` annotations do three things:

1. **Name the producer and consumer** — the compiler verifies that `TradingEngine` does in fact write to `riskLimits` in its declared window, and that `RiskChecker` does in fact read from it. A mismatch is a compile error. Anonymous shared memory — a location written by nobody-in-particular and read by nobody-in-particular — cannot be expressed.

2. **Declare staleness tolerance** — `maxStaleness = "1ms"` tells the compiler that `RiskChecker` tolerates reading a value up to 1 ms old. The compiler translates 1 ms to tick count, verifies the gap between `TradingEngine`'s write window and `RiskChecker`'s read window is ≤ that count, and emits a compile error if `TradingEngine` might not update within the tolerance. The runtime logs actual staleness to the observability channel on every read.

3. **Enable cross-circuit optimisation** — knowing both producer and consumer, the compiler can place their windows on cores that share a cache domain, reducing coherency latency. Without the annotation, the compiler places windows by availability. With it, the compiler optimises placement specifically for that handoff pair.

`@Handoff` is not required — a `Channel<T>` subscription already declares the connection. `@Handoff` is the programmer's way of asserting stronger properties — named ownership, staleness bounds, placement hints — that the compiler then verifies. It is an annotation that adds proof obligations, not one that relaxes them.

### Cryptographic Circuit Identity

Every compiled circuit manifest is signed with the organisation's private key before deployment. The signature covers the circuit's declared timing, lifetime types, channel subscriptions, and `@Handoff` declarations — the complete proof payload. The runtime verifies the signature before slotting the circuit into the dispatch table. An unsigned manifest, or one signed by an unknown key, is rejected at the dispatch boundary before any window is allocated, before any channel subscription is registered.

`@Handoff` extends this naturally. A handoff between two circuits is only permitted if both manifests are signed by keys within the same declared allocation:

```java
@Handoff(tier = Session, from = "TradingEngine", to = "RiskChecker",
         maxStaleness = "1ms", org = "AcmeTradingCo")
val riskLimits: Session<RiskLimitTable>
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

**The invariant the compiler maintains:**

> Every circuit starts exactly at its declared tick. Every circuit finishes before or at its budget. Every handoff is proven to complete before the reader's window opens. These are theorems, not assumptions.

The hardware clock is the only synchronisation primitive in the entire system. It was always there. It was always running. No OS ever listened to it.

---

## Hardware Backing

### The Runtime Resolves Physical Backing — The Programmer Does Not

The programmer writes `Channel<NicFrame>` and calls `in.read()`. There is no DMA in the standard library. There is no `mmap`, no `ioctl`, no driver call, no interrupt registration. The channel is a channel.

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

The clock-aware channel model makes this structural. An SPI sensor becomes:

```java
val sensor = Channel.from(Device.SPI0, clock: 1.MHz5)
sensor.subscribe(reading -> process(reading))
```

The compiler resolves from the `Device.SPI0` descriptor: the DREQ line number, the DMA channel, the clock divider from the declared rate and `cpu_model`, the CPOL/CPHA mode, the chip-select timing, the bus address translation, the cache flush requirements, and the IRQ registration. The programmer declared what they needed — a 1.5 MHz SPI channel from device SPI0. The compiler translated that declaration into every hardware detail the runtime needs. No oscilloscope. No logic analyser. No three days debugging CPOL/CPHA mode mismatch. The device descriptor carries the mode; the compiler applies it; the compiler verifies the type matches the SPI frame size. If it compiles, the wiring is correct.

### The Runtime Is the Type Enforcer

Hardware-incorrect code does not compile. Not because a linter ran after the fact. Because the type system encodes the hardware model: `Channel<T>` is typed, so reading the wrong type is a type error. Subscription lists are compile-time verified, so reading an undeclared channel is a compile error. Cycle budgets are verified by the proc-macro against `llvm-mca`, so exceeding the budget is a compile error. The type system is the hardware contract. The compiler is the enforcer. The runtime confirms what the compiler already proved.

### Generation 1 to Generation 2 — Closing the Loop

Generation 1 ships a working system: the language, the compiler, the runtime, the kernel circuits — all written in the language — plus ~500 lines of Assembly stubs for boot and hardware initialisation. The Assembly is not hidden; it is the declared boundary between what the language can express and what requires raw instruction sequences.

Generation 2 erases that boundary. The compiler, now running on Generation 1, rewrites the Assembly stubs as declared `Channel<T>` functions. The boot sequence becomes a declared function with a declared timing contract. Hardware initialisation becomes a sequence of channel writes. The ~500 lines of Assembly become zero. The system is fully expressed in the language that proved hardware-correctness of everything else.

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
  NicCircuit → Channel<EthernetFrame> → PacketProcessor → ...
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

The ADRs in this repository record implementation decisions for a near-term Rust prototype: proc-macro annotations, `system.cap`, `Channel<T>`, `cargo-timeslice`. That prototype is a research vehicle — a way to validate the scheduling model, the channel model, and the verification approach on existing hardware using existing toolchain infrastructure. It is not the OS described in this paper.

The channel-based I/O model (ADR-0010) is the first instantiation of the unified I/O abstraction. The unified system configuration (ADR-0005) is the first instantiation of the hardware-model declaration. The CPU partition model (ADR-0002) is the first instantiation of the static dispatch table. These concepts carry forward directly into the runtime.

Each ADR is a step toward understanding. This paper is the runtime destination.

---

*Part of the clock-aware programming series. See [Paper I: Clock-Aware Programming](01-clock-aware-programming.md) for the core primitive. See [Paper II: The Language](02-language.md) for the language definition. See [Paper IV: Hardware Architecture Implications](04-hardware-architecture.md) for the silicon consequences.*
