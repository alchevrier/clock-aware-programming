# Paper II: The Language
#### Alex Chevrier — chevrier.alex@gmail.com

> When every state is declared, the language ceases to be expressive and becomes a proof system.

This paper follows from [Paper I](01-clock-aware-programming.md). It defines the clock-aware language from first principles — four rules, derived entirely from the hardware model. The OS and runtime consequences of this language are in [Paper III](03-os-and-runtime.md). The hardware consequences are in [Paper IV](04-hardware-architecture.md).

---

## The Starting Point

Paper I establishes that when every operation's timing is declared and compiler-verified, the machinery that compensates for not knowing when things happen — schedulers, locks, RCU, memory barriers, GC, interrupt dispatch — becomes provably unnecessary. The question this paper answers is: what does the language look like once you build from that foundation rather than adding it on top?

The answer is a language with four rules instead of four thousand pages — designed from the declarations down, not from a syntax up. If it compiles, it is hardware-correct. Not by convention. By construction.

---

## One Language

The language comes first. Not Rust with annotations bolted on. Not an existing language extended with timing attributes. A new language whose entire surface is the clock-aware model — designed from the declarations down, not from a syntax up.

The implementation path follows two generations:

- **Generation 1** — the language is written and the compiler targets the language itself. The unavoidable hardware boundary — system calls, boot, hardware initialisation — is covered by a thin layer of Assembly stubs (~500 lines). Everything above the stubs is written in the language.
- **Generation 2** — the compiler rewrites those Assembly stubs in the language itself, using `Channel<T>` declarations to express what the stubs previously expressed in raw instruction sequences. The language becomes fully self-hosting. No assembly. No Rust. No foreign layer.

The stubs in Generation 1 are not a concession — they are the hardware boundary that cannot be abstracted away without first having a language capable of expressing it. Generation 2 is when the language proves it can.

### Four Rules

Today's systems languages — C++, Rust — require the programmer to hold a large mental model simultaneously: ownership, lifetimes, borrow rules, unsafe boundaries, memory ordering semantics, async runtimes, move semantics, reference types. The complexity is not the price of expressiveness; it is the price of building on a foundation that does not know when things happen. Every rule in Rust's ownership system exists because the compiler does not know the relative timing of two accesses — so it must conservatively restrict them.

A language where every access cycle is declared has none of this uncertainty. The rules collapse to four:

| Rule | What it declares |
|---|---|
| **Clock annotation** | When this operation runs: core, cycle, budget |
| **Lifetime type** | Which memory tier this value lives in: L1, L2, declared heap |
| **Channel** | All I/O: every hardware signal, every inter-circuit boundary, unified |
| **Exhaustive match** | All states: every branch explicitly declared, including the intent to ignore or default |

These four rules capture everything the compiler needs to prove hardware-correctness end-to-end. There is no fifth rule. There is no escape hatch. There is no `unsafe`. There is no `&`, `*`, `&&`, `&mut`, `Box`. There is no `new` — because allocation is a lifetime declaration, not a constructor call. There is no move semantics — because memory lifetime is declared, not inferred.

The exhaustive match rule is stricter than conventional pattern matching. There is no silent wildcard. A `_` arm that discards unhandled cases is a compile error — not because wildcards are forbidden, but because the intent must be declared explicitly:

```java
switch (event) {
    case Fill(Fill f)     -> handleFill(f);
    case Cancel(Cancel c) -> handleCancel(c);
    case Reject           -> ignore;         // explicit: this case is intentionally discarded
    default               -> defaultLog;     // explicit: all other cases go to declared handler
}
```

`ignore` and `defaultLog` are not special keywords — they are named handlers that must be declared in the subscription list. A match arm that names no handler, or that silently falls through, is a compile error. Every state the programmer does not handle explicitly is a state they must explicitly decide to ignore or route to a named default. The compiler enforces that the decision was made, not that it was correct.

### Lifetime Types — The Memory Tier System

Lifetime types are the second rule. They are type keywords — the same way `int` declares a value is an integer, `Permanent` declares a value lives for the programme's lifetime in pinned DRAM. The compiler maps the declaration to the correct physical tier, pins the value there, and reclaims it at the declared boundary. No `malloc`. No `free`. No GC. The lifetime declaration is the memory contract.

| Lifetime | Scope | Maps to | Notes |
|---|---|---|---|
| `Register` | Single expression | Registers only | Never touches memory; guaranteed, not hinted |
| `Ephemeral` | Function scope | L1, pinned | Intermediate results, parsing scratch, message buffers |
| `Task` | Current cycle window | L1/L2, pinned for window duration | Active transaction state, channel read buffers |
| `Session` | Lifetime of a connection or run | DRAM, resident | Connection state, compiled code cache, configuration |
| `Permanent` | Programme lifetime | DRAM, pinned — never paged | Model weights, read-only lookup tables, static data |
| `Cold` | Indefinite, infrequently accessed | NVMe, demand-loaded | Historical data, audit logs — loaded at declared cycle, no surprise page faults |

`Cold` is the tier no managed language can express. Java, Go, Python have no concept of "load this at a declared cycle from storage." In the clock-aware model, a `Cold` access is declared, its load cycle is verified against the NVMe latency in `system.cap`, and the compiler rejects declarations whose timing budget is insufficient. Storage latency becomes a compile-time constraint, not a runtime surprise.

**The compiler guarantees L1 utilisation.** `system.cap` declares the L1 cache capacity of the target core. The compiler sums the declared footprints of every `Ephemeral` and `Task` value in a circuit's active window — every collection size, every channel buffer, every intermediate value. If the total exceeds the L1 capacity declared for the assigned core, the compiler rejects the programme with a precise error: which values, how many bytes each, and by how much the L1 is exceeded. If it fits, the compiler emits a proof that the circuit's hot path working set is entirely resident in L1 for the duration of its window — no evictions, no reloads, no cache misses. This proof is part of the manifest. The trace unit validates it at runtime: a cache miss on the hot path of a circuit with a passing L1 proof is a `Channel<ExecutionPlanViolation>` signal, flagging either a hardware anomaly or a compiler model mismatch against `system.cap`.

The guarantee is not "probably L1-resident" or "L1-resident under normal conditions". It is unconditional: the working set fits by construction, and any deviation is immediately observable and attributed.

The runtime extends this statically: it tracks L1 utilisation live via the atom stream. Each atom carries the actual cache line occupancy of the circuit that just executed. The `ObservabilityCircuit` aggregates this into a per-core L1 pressure map updated every tick. When a core's L1 pressure consistently falls below a threshold — because circuits are finishing with significant working-set slack — the runtime has two responses:

1. **Adjust the execution plan.** The runtime signals the `AdaptationCircuit` via `Channel<L1Advisory>`. The adaptation circuit can recommend promoting a `Session`-tier value to `Ephemeral` for that core — pre-loading it into L1 ahead of the next window that needs it — reducing effective access latency without the programmer declaring it. The compiler's static proof is the floor; the runtime's promotion is a safe tightening within it.

2. **Switch `clock_model`.** A core whose L1 is consistently underutilised is a core whose circuits are consistently finishing early. The runtime can downgrade its `clock_model` from `Performance` to `Balanced` — reducing frequency and power — with the proof that the circuits' declared windows still close in time at the lower frequency. The switch is not a guess. It is derived from the L1 utilisation data and the compiler's known instruction count for each circuit on that core.

Both of these are *reactive* responses. The runtime has a third, more powerful capability: **speculative clock model pre-adjustment**. Because the dispatch table is known in full for the next N ticks — it is a compile-time theorem — the runtime can look ahead and pre-transition a core's `clock_model` before a burst arrives, not in response to it.

The concrete case is a DMA burst from the NIC. The runtime knows from the dispatch table that in 8 ticks, the `NicCircuit` will deliver a `Channel<EthernetFrame>` burst and the `PacketProcessor` circuit will begin executing on core 3. Core 3 is currently in `Efficiency` mode. The runtime pre-transitions core 3 to `Performance` at tick N−2 — two ticks before the burst window opens — so the core is at full frequency and full pipeline depth when the first frame arrives. The `PacketProcessor` never executes a single cycle at reduced frequency. The transition cost is paid in advance, in a gap the dispatch table already has scheduled.

```
  tick N-8  NicCircuit declares frame burst incoming
  tick N-2  runtime pre-transitions core 3: Efficiency → Performance
                (dispatch table showed burst at N, transition latency = 2 ticks)
  tick N    PacketProcessor executes — core already at full speed
  tick N+k  burst complete — runtime schedules Efficiency re-entry
```

This is the general pattern: the dispatch table is the lookahead. Every future burst, every future handoff, every future DMA completion is visible in it. The runtime uses this knowledge to pre-condition hardware state — clock frequency, L1 pre-population, core affinity migration — so that when the window opens, the hardware is already in the correct state. There is no warmup cost. There is no reactive lag. The preparation is as provable as the execution.

The reason this works — and the reason it is not speculation in the conventional sense — is that the runtime knows from the manifest exactly which circuit subscribes to which channel. The channel subscription graph is a compile-time fact, not a runtime discovery. When the `NicCircuit` declares it produces to `Channel<EthernetFrame>` and the `PacketProcessor` declares it subscribes to that channel, the runtime has a complete causal map: NIC DMA completion → frame on channel → `PacketProcessor` window opens. Every hop in that chain is declared. None of it is inferred at runtime. The runtime is not predicting what will happen — it is reading a theorem that the compiler already proved. Pre-conditioning the hardware for that chain is executing a known consequence, not betting on an uncertain future.

A conventional OS has no equivalent knowledge. It sees a file descriptor — an integer. File descriptor 7 might be a TCP connection, a pipe, a regular file, or a Unix socket. The OS does not know. The application knows, but it never told the OS. The result is that the OS cannot pre-condition anything: it reacts to a `read()` syscall, discovers at that moment what fd 7 is connected to, hands off to the right driver, and waits. The hardware was idle during the discovery. The core was at whatever frequency it happened to be at. The cache contained whatever happened to be there. Every step is reactive because the information was never declared.

This is the deepest sense in which the clock-aware model is **self-aware**: the runtime knows who uses what, exactly as an electronic circuit knows its routing. In an electronic circuit you do not put a signal into a wire and hope the right component picks it up. You design a routing table — traces, buses, demultiplexers — that is fixed at PCB design time. The signal reaches exactly the right component in exactly the right number of nanoseconds because the path was declared before the board was etched. The efficiency is structural, not emergent.

The clock-aware model applies this to software. `Channel<EthernetFrame>` is a trace on the PCB. The `NicCircuit` is the component that drives it. The `PacketProcessor` is the component that receives it. The routing table is the channel subscription graph in the manifest. The compiler is the PCB designer. The runtime is the board running — no routing decisions at runtime, because the routing was declared at design time. That is why it is efficient. That is why it is provable. That is why a conventional OS, which makes routing decisions at runtime from file descriptors it does not understand, can never achieve the same guarantees.

The reason conventional systems cannot do this is not a missing feature — it is a structural disconnection. A DMA device always knew everything it needed to say: when it would have data ready, how much, at what rate, on what bus. The NIC knows it is about to deliver 64 Ethernet frames. The DMA controller knows the source address, the destination, the transfer size. The DREQ signal knows when the peripheral is ready. All of that information existed at hardware design time. And the application programmer always knew what circuit would consume the data: the packet processor, the parser, the inference engine. That too was known at design time.

But nothing connected the two. The OS sat between them, ignorant of both. The DMA completed and wrote to a memory location. The scheduler, in a quiescent state, eventually noticed. The application was woken up. It called `read()`. The OS discovered — at that moment, for the first time — that this was a network socket and routed to the NIC driver. The data moved. The consumer ran. Every step was a runtime discovery of information that was always known, just never declared. The system was not slow because the hardware was slow. It was slow because the information flow was disjointed: hardware knew its end, software knew its end, and the OS bridged them at runtime with no knowledge of either.

The clock-aware model closes the gap by making the connection at compile time. The DMA device's timing is declared in `system.cap`. The channel it produces to is declared in the driver circuit. The consuming circuit is declared in the subscription graph. The compiler wires them together — device timing to channel to consumer window — and the runtime executes the resulting proof. There is no quiescent state to emerge from. There is no runtime discovery. The NIC's DREQ fires, the DMA transfers, the channel is written, the consumer's window opens — all at declared ticks, all proven in advance, all connected end to end before the first packet arrived.

### Surface

The language surface is Java-like in familiarity but Kotlin-like in concision: `fn` for functions, return types inferred, `val` for immutable bindings. No `void`. No explicit `return` for single-expression functions. Learnable in days.

A function is a circuit. Its output is a signal — the value the circuit produces. That signal is the return value, not a parameter:

```java
@Timeslice(core = 2, cycle = "4ns")
fn parsePrice(in: Channel<RawMessage>): Channel<Price> =
    Channel.of(extract(in.get(consumerIndex)))
```

This makes composition plain function application — exactly as in circuit diagrams where you connect the output wire of one stage to the input wire of the next:

```java
@Timeslice(core = 2, cycle = "4ns")
fn pipeline(raw: Channel<RawMessage>): Channel<BookSnapshot> =
    updateBook(parsePrice(raw))   // raw → prices → book
```

Multiplexing is the same: the returned channel is a value the caller can pass to multiple consumers. Each consumer calls `get` with its own index. No special syntax. No actor framework. No pub/sub runtime. Function calls and return values.

Passing `out` as a parameter — the imperative style — is not expressible. A function does not take its output wire as input. It produces a signal. The type signature is the complete interface: what comes in, what comes out, when it runs.

The programmer writes the logic. The annotations declare the contract. The compiler proves the contract is consistent with the hardware model in `system.cap`. There is no runtime that the programmer reasons about separately from the language — the runtime enforces the same contract the compiler proved.

### The Compiler Is ALU-Aware

Proving that a declared budget is honourable requires the compiler to know the ALU cost of every operation it compiles. A declaration of `@Timeslice(cycle = "4ns")` is meaningless unless the compiler can count the cycles the function's instruction sequence actually consumes on the target microarchitecture.

The compiler maintains a per-target **ALU cost table**: the cycle count for each instruction or micro-operation on the target execution units, derived from the microarchitecture model in `system.cap`. When it compiles a function, it walks the instruction sequence and accumulates a cycle budget:

```
fn parsePrice — @Timeslice(cycle = 4ns)

  instruction             execution unit   cycles
  ──────────────────────  ───────────────  ──────
  channel.get()           load port        1
  status field extract    ALU              1
  branch (predicted)      branch unit      0
  extract(msg)            ALU × 3          3
  Channel.of()            store port       1
                                          ──────
  hot path total                           6 cycles  →  1.5 ns @ 4 GHz

  declared budget:  4 ns  (16 cycles @ 4 GHz)
  hot path cost:    6 cycles
  margin:           10 cycles  ✓  budget honoured
```

If the accumulated cost of the hot path exceeds the declared budget, the compiler rejects the programme with a precise error: which operations consumed which cycles, and by how much the budget was exceeded. This is not a warning. It is the same class of error as a type mismatch.

The ALU cost table is not a heuristic. It is derived from the microarchitecture descriptor in `system.cap` — the same source the runtime uses to verify timing at slot time. The programmer declares `system.cap = skylake-4ghz`; the compiler loads the Skylake execution port latencies; the budget check uses those exact numbers. On a different target, the same source code with the same annotation may require a larger budget — the compiler will say so.

This is what makes `@Timeslice` a proof obligation rather than a hint. The compiler is not estimating whether the function will probably fit. It is calculating whether it provably fits, from first principles, against the declared hardware model. If it does, the timing annotation is a theorem. If it does not, it is a compile error.

The target in `system.cap` can be chosen for compliance purposes as well as for hardware accuracy. A system targeting automotive safety certification declares `compliance = ISO-26262-ASIL-D`; a medical device declares `compliance = IEC-62304-Class-C`; an avionics system declares `compliance = DO-178C-Level-A`. The compiler loads the corresponding constraint profile — tighter budget margins, mandatory worst-case execution time (WCET) bounds, required proof artefact format — and validates against those requirements in addition to the hardware model. The timing proof the compiler produces is not just internally consistent; it is in the format the certification body requires.

This makes the compiler the certification tool. There is no separate static analyser, no WCET tool bolted on after the fact, no manual timing analysis in a spreadsheet. The same compilation step that proves hardware correctness produces the compliance artefact. A programme that compiles against `compliance = DO-178C-Level-A` is, by construction, a programme whose timing behaviour is provable to the required standard.

### The Compiler Does the Hardware's Job

When the compiler knows ALU costs, timing constraints, register availability, and execution port utilisation — all declared in `system.cap` — it can perform every optimisation that hardware does speculatively at runtime, but do it statically, deterministically, and provably at compile time. The hardware then just runs.

**Instruction reordering.** The compiler analyses data dependencies across the entire function body and reorders instructions to fill execution unit latency gaps. No out-of-order engine needed at runtime — the order is already optimal in the binary.

**Wave pipelining.** The compiler schedules instruction waves across pipeline stages such that a new wave starts before the previous wave commits. This is what the hardware's pipeline does implicitly; the compiler makes it explicit and provable, independent of the hardware's speculation machinery.

**Register collapsing.** With 30 architectural registers available in the target model and full visibility of the function's value lifetimes, the compiler assigns registers greedily — values that expire before a new value is needed reuse the same register. No spill to the stack. No reload. The register file is fully allocated at compile time with zero waste.

**Branch elimination.** Branches are truth tables, not jumps. The compiler analyses every branch in the hot path and emits conditional move sequences — or pipelined table lookups across cores where the branch condition is the selector — replacing the branch with arithmetic. There is no branch predictor to warm up. There is no misprediction to recover from. The condition evaluates and the correct value is selected in a declared number of cycles.

**Speculative execution — safe because deterministic.** The compiler can speculatively evaluate both sides of a branch, because it knows statically which paths are reachable and what their costs are. This is safe — no UB, no speculation barrier, no Spectre — because the compiler's speculation is not hardware speculation. It is a compile-time decision that has already been proven correct. The hardware executes the precomputed result.

**Profile-guided optimisation (PGO).** When execution traces are available (from the trace unit — see Paper III), the compiler ingests them as input to the instruction ordering and register allocation passes. The optimised binary is derived from real execution data, not profiling heuristics. The trace is a proof input, not a suggestion.

The consequence is that the CPU the clock-aware model implies has no need for an out-of-order engine, a branch predictor, a speculative execution buffer, a register rename table, or a return address stack. Those mechanisms exist to recover at runtime from decisions the compiler was never asked to make. Ask the compiler to make them, and the recovery machinery is unnecessary. What remains is a deep execution pipeline, large register file, wide issue ports — all of them doing computation, none of them doing speculation. Every transistor computes.

### Immutable Arguments

All function arguments are immutable by default. Data flow is explicit: a value flows into a function through a declared input channel, is transformed, and flows out through a declared output channel. There is no shared mutable state because there is no mechanism to express it. Side effects are channel writes — declared, typed, subscription-checked.

This is not a restriction. It is the observation that a timed function has no shared mutable state by construction — it has inputs, outputs, and internal state that does not escape its cycle boundary. The language matches the model.

### Hot and Cold Paths — Declared Priority Within a Function

A timed function has a declared cycle budget. But control flow within that function may diverge: the main pipeline processes the common case, while other paths handle observability, error logging, metrics emission, or background maintenance. If those secondary paths run in the same cycle window as the hot path, they consume budget that the main pipeline needs.

The type system detects this structurally. A path that uses a `Cold<T>` type is automatically inferred as cold — the type carries the path annotation. The programmer may also declare it explicitly:

```java
@Timeslice(core = 2, cycle = "4ns")
fn parsePrice(in: Channel<RawMessage>): Channel<Price> {
    val raw = in.get(consumerIndex);

    return switch (raw.status) {
        case Ok(msg)    -> Channel.of(extract(msg));          // hot path
        case Malformed  -> @Cold { metrics.put(errorIndex, raw); Channel.empty() }
        default         -> ignore;
    };
}
```

Circuit equivalent:

```
                         ┌──────────────────────────────────────────────────┐
                         │  parsePrice — @Timeslice(cycle = 4ns)            │
                         │                                                  │
  Channel<RawMessage> ──►│ in.get()    ┌────────────┐                      │
  (input port)           │             │   switch   │ Ok                   │
                         │             │  raw.status├──────► extract() ───►│──► Channel<Price>
                         │             │            │        (hot path)    │    (output signal)
                         │             │            │        ← 4ns budget →│
                         │             │            │                      │
                         │             │            │ Malformed            │
                         │             │            ├──────────────────────┼──► @Cold deferred
                         │             │            │  metrics.put()       │    (separate window,
                         │             │            │  Channel.empty()     │     not in budget)
                         │             │            │                      │
                         │             │            │ default              │
                         │             │            ├──────► ignore        │
                         │             └────────────┘                      │
                         │                                                  │
                         │  ←────────── 4ns budget ────────────────────►   │
                         │              hot path only                       │
                         └──────────────────────────────────────────────────┘
                                              │
                                    @Cold window (lower priority,
                                    allocated separately by compiler,
                                    structurally cannot delay hot path)
```

The `@Cold` annotation on a branch tells the compiler two things:
1. **This path does not count against the hot path's declared budget.** The 4 ns budget is the budget for the `Ok` arm. The cold arm is allocated a separate, lower-priority window.
2. **If this path's execution could delay the hot path, it is a compile error.** The compiler verifies that cold paths cannot block the cycle boundary of the hot path — they are structurally deferred, not merely hinted.

Using a `Cold<T>` type anywhere in a branch is sufficient for the inference — the programmer does not need to annotate every cold branch manually if the type already declares it. A `Cold<Metric>` channel write is self-evidently not on the hot path; the compiler treats the entire branch as cold without an explicit `@Cold`.

This is what replaces `__builtin_expect` and `[[likely]]`/`[[unlikely]]` hints in C++. Those are hints the compiler may or may not respect. In the clock-aware model, path priority is a type-level property with a timing contract: the compiler proves the cold path cannot violate the hot path's budget, or rejects the programme. Observability, logging, and background maintenance become structurally isolated from the main pipeline — not by discipline, but by construction.

### Array-Based Standard Library

Every collection in the standard library is array-backed. There are no linked lists. No tree nodes allocated on the heap. No hash map buckets chained through pointers. No `Vec<Box<T>>`. The standard library does not offer these structures — not because they are forbidden by a rule, but because they are inexpressible without `new` and pointer indirection, which do not exist.

| Conventional | Clock-aware stdlib |
|---|---|
| `LinkedList<T>` — pointer per node, cache-hostile | `ArrayDeque<T>` — contiguous, prefetcher-optimal |
| `HashMap<K,V>` — heap-allocated buckets, pointer chase | `FlatMap<K,V>` — sorted array, binary search, L1-resident |
| `BTreeMap<K,V>` — tree nodes, pointer chase | `SortedArray<K,V>` — contiguous, declared size |
| `Vec<Box<T>>` — pointer indirection per element | `Array<T>` — inline elements, declared capacity |

Every access pattern the programmer can express in the standard library is one the hardware prefetcher can predict: stride access into a contiguous array. Cache miss latency from pointer chasing is not a performance concern the programmer must avoid — it is a construct the language does not provide. The hardware-friendliness is structural, not disciplinary.

Custom collections can be defined by three declarations:

```java
collection PriceTable {
    val size = 1024;
    fn writeAccessPattern(symbol: Symbol) = symbol.id & 0x3FF
    fn readAccessPattern(symbol: Symbol)  = symbol.id & 0x3FF
}
```

- **`size`** — the declared capacity, in elements. The compiler knows the memory footprint at build time, verifies it fits within the declared tier (L1, L2), and rejects declarations that exceed it.
- **`writeAccessPattern`** — a pure function from write key to index. Called by the producer to locate the slot to write into.
- **`readAccessPattern`** — a pure function from read key to index. Called by the consumer to locate the slot to read from. The key type may differ from the write key.

The compiler statically analyses both functions to verify they are branch-free, have no allocations, and produce indices within `[0, size)`. Read and write keys may be different types — the compiler enforces that each access site passes the correct key to the correct pattern.

This replaces hash functions, tree traversals, and associative containers at once. The programmer declares how keys map to positions; the compiler proves the mapping is hardware-correct. There is no runtime hash table implementation, no collision resolution, no load factor heuristic. The access pattern is a compile-time contract, not a runtime algorithm.

The consequence is that standard data structures fall out as special cases. A SPSC ring buffer is the clearest example — producer and consumer have different keys and independent access patterns:

```java
collection Spsc<T> {
    val size = 64;
    fn writeAccessPattern(producerIndex: Long) = producerIndex & (size - 1)
    fn readAccessPattern(consumerIndex: Long)  = consumerIndex & (size - 1)
}
```

The producer passes `producerIndex`; the consumer passes `consumerIndex`. The access pattern for both is the same arithmetic, but the keys are independent — the compiler enforces that the producer never calls `readAccessPattern` and the consumer never calls `writeAccessPattern`. No head/tail pointer pair managed manually. No atomic increment. No CAS loop. The index arithmetic is the access pattern — declared, compiler-verified, branch-free by construction. Coordination is structural: the producer writes at its declared cycle; the consumer reads at its declared cycle; the timing proof is what prevents the race, not a lock.

A FIFO (queue) makes the asymmetry explicit — tail advances on write, head advances on read, and they are independent keys:

```java
collection Fifo<T> {
    val size = 64;
    fn writeAccessPattern(tail: Long) = tail & (size - 1)
    fn readAccessPattern(head: Long)  = head & (size - 1)
}
```

A LIFO (stack) is different again — both read and write use the same index expression (the top pointer), but the key moves in opposite directions: the caller increments it before a write and decrements it before a read:

```java
collection Lifo<T> {
    val size = 64;
    fn writeAccessPattern(top: Long) = top & (size - 1)   // caller increments top first
    fn readAccessPattern(top: Long)  = top & (size - 1)   // caller decrements top first
}
```

The collection does not manage the key — the caller holds it and passes it on each access. The collection declares only the mapping from key to slot. This means the collection has no internal state: no head, no tail, no top field stored anywhere. The key lives in the caller's declared cycle scope, reclaimed at the cycle boundary like any other value.

The entire collection API is therefore two operations:

```java
collection.put(key, value)   // resolves slot via writeAccessPattern(key), writes value
collection.get(key)          // resolves slot via readAccessPattern(key), returns value
```

No `push`. No `pop`. No `enqueue`. No `dequeue`. No `peek`. Every data structure in the language, regardless of its access discipline, exposes the same two-operation interface. The discipline — FIFO, LIFO, ring, table — is encoded in what key the caller passes, not in the API. `put` and `get` are the only verbs the programmer needs to know.

This is the deeper unification: `Channel<T>` itself is a collection with a declared access pattern. The channel model does not sit above the collection model — it is an instance of it. A `Channel<T>` is simply a collection whose `writeAccessPattern` and `readAccessPattern` happen to be the ring-buffer masking arithmetic, and whose keys — the producer and consumer indices — are managed by the runtime at the cycle boundary.

Every inter-function data path in the system, whether called a channel, a ring buffer, a queue, a stack, or a lookup table, is the same thing: a declared size and a pair of declared access patterns, accessed via `put` and `get`. The word "channel" is not a special construct — it is a named convention for a collection whose access pattern encodes ordered delivery. If the programmer wanted, they could declare it themselves. The stdlib `Channel<T>` is provided as a convenience, not as a primitive.

The compiler sees one construct. The hardware sees one thing: a stride into a contiguous array.

**Every standard library function declares its own cycle budget.** This is mandatory — a stdlib function without a declared `@Timeslice` does not compile into the standard library. The declaration is per-target: `put` on `FlatMap<K,V>` against a Skylake target at 4 GHz costs a different number of cycles than the same call against a Cortex-A72. The stdlib ships a budget table per supported target, derived from the same ALU cost model the compiler uses.

When the programmer calls a stdlib function inside a timed `fn`, the compiler substitutes the stdlib function's declared budget for the target into the caller's accumulated cycle count. The programmer does not need to look up the cost manually — the compiler propagates it:

```
fn parsePrice — @Timeslice(cycle = 4ns) — target: skylake-4ghz

  operation                       source    cycles
  ──────────────────────────────  ────────  ──────
  channel.get()                   stdlib     1
  status field extract            ALU        1
  FlatMap.get(symbol)             stdlib     3     ← binary search, 1024 elements
  extract(msg)                    ALU        3
  Channel.of()                    stdlib     1
                                            ──────
  hot path total                             9 cycles  →  2.25 ns @ 4 GHz

  declared budget: 4 ns (16 cycles)
  hot path cost:   9 cycles
  margin:          7 cycles  ✓
```

The stdlib budget entries are labelled `stdlib` in the compiler's output — distinct from user ALU operations — so the programmer can see exactly which stdlib calls dominate their budget and replace them with cheaper alternatives if needed.

This has a second consequence: the standard library cannot contain a function whose worst-case cost is unbounded. A function whose cycle count depends on runtime input size — a sort, a full linear scan, an unbounded iteration — cannot be declared with a fixed budget and therefore cannot be in the stdlib. The stdlib is, by construction, a collection of functions with statically-bounded costs. Unbounded operations must be expressed as circuits with declared windows and explicit `@Cold` deferred paths, not as stdlib calls on the hot path.

### A Class Is a Circuit with an ALU

In conventional languages a class is a bundle of data and methods — an organisational unit with no hardware meaning. In the clock-aware language a class is a circuit: it has declared state (registers), declared operations (the ALU — the combinational logic that transforms inputs to outputs), and a declared timing contract.

```java
@Timeslice(core = 2, cycle = "4ns")
class OrderBook {
    Permanent PriceTable bids;        // declared state — lives in declared memory tier
    Permanent PriceTable asks;

    fn update(in: Channel<Fill>): Channel<BookSnapshot> =
        snapshot(apply(in.get(consumerIndex), bids, asks))   // ALU: transform input to output

    fn bestBid(): Price = bids.get(topKey)   // combinational read — output signal pin
}
```

Circuit equivalent:

```
                    ┌─────────────────────────────────────────────┐
                    │               OrderBook                      │
                    │           @Timeslice(cycle = 4ns)            │
                    │                                              │
  Channel<Fill> ───►│ update()    ┌──────────┐                    │
  (input port)      │             │   ALU    │                    │
                    │    ┌───────►│ apply()  ├──────────────────► │──► Channel<BookSnapshot>
                    │    │        │ snapshot()│                   │    (output signal)
                    │    │        └──────────┘                    │
                    │    │                                         │
                    │  ┌─┴────────────────────┐                   │
                    │  │   Register File       │                   │
                    │  │  Permanent PriceTable │                   │
                    │  │  bids ░░░░░░░░░░░░░  │                   │
                    │  │  asks ░░░░░░░░░░░░░  │                   │
                    │  └──────────────────────┘                   │
                    │                                              │
                    │ bestBid()                                    │
                    │ (output pin) ───────────────────────────────►│──► Price
                    │                                              │    (exposed signal)
                    └─────────────────────────────────────────────┘
                           ▲
                    setter pin (control input)
                    e.g. setTickSize(size: TickSize)
                    drives into register file at next cycle boundary
```

`update()` is the critical path through the ALU — declared at 4 ns, verified by the compiler against `llvm-mca`. `bestBid()` is a combinational read from the register file — no ALU, no cycle budget, always current. The setter is a control input that drives a new value into the register file at the next cycle boundary — no shared memory access from the outside, just a pin.

The class's state is its register file — values that persist across cycle boundaries at their declared tier. The methods are its ALU — combinational logic that takes the current state and an input and produces an output signal. The `@Timeslice` annotation is the timing constraint on the critical path through that logic, exactly as a timing constraint in Vivado applies to the critical path through a clocked block.

This is not a metaphor. A hardware register holds a value across clock edges. A `Permanent PriceTable` holds a value across cycle boundaries. A hardware ALU computes a result in one clock cycle from registered inputs. A timed method computes a result within its declared budget from declared-tier inputs. The correspondence is structural.

The implication is that a system described in this language is a netlist of circuits: each class is a clocked block, each function call is a wire connecting output signals to input ports, each channel is a registered signal crossing a clock boundary. The compiler is the synthesis tool. `system.cap` is the constraint file. The binary is the bitstream.

A getter on a class is an exposed output signal — a pin on the circuit's interface that other circuits can read to parametrise their own behaviour. A setter is a control input pin — a signal another circuit drives to configure this one. Neither implies shared mutable state. The getter reads from the class's declared register file and returns the current value as a signal. The setter drives a value into the register file at the next declared cycle boundary. Both are typed, subscription-checked, and timing-verified by the compiler.

This is exactly how hardware IP blocks expose their configuration interface: an AXI-Lite register map is a set of setters; a status register is a set of getters. The clock-aware class models the same thing in software. A circuit that needs to know the current best bid does not share memory with the `OrderBook` — it reads the `OrderBook`'s `bestBid()` output signal at its declared cycle. No lock. No cache-line ping-pong. No race condition. The signal's timing is declared; the compiler proves the reader's cycle comes after the writer's cycle.

The design rule follows directly:

| Use | When |
|---|---|
| **`class`** | The circuit has internal state that persists across cycle boundaries and whose output signals parametrise other circuits. Output depends on accumulated state + current input. |
| **`fn`** | The circuit's output depends only on its current input signal. No persistent state. Pure transformation: input in, signal out. |

A class is a stateful circuit — a clocked block with a register file. A function is a combinational circuit — a wire with logic on it. In hardware terms: if you need a flip-flop, use a class. If you need only gates, use a function. The compiler enforces the distinction: a `fn` cannot hold state across cycle boundaries; only a `class` field declared with a lifetime type persists beyond the current cycle.

### No Unsafe Escape Hatch

The absence of `unsafe` is the strongest claim. In Rust, `unsafe` exists because the language model cannot express all valid programs without it: hardware access, FFI, pointer arithmetic. In the clock-aware language, all hardware access is through `Channel<T>`. There is no raw pointer because there is no need for one — the channel model covers every hardware interaction, and the compiler knows the full channel graph from `system.cap`. There is nothing that `unsafe` would permit that the type system cannot already express.

A program that cannot be expressed without `unsafe` is a program that has not declared its timing. The solution is to declare it, not to escape the model.

---

## Fully Hardware Compliant

"Runs on hardware" is a description of every program ever written. It means nothing.

"Fully hardware compliant" means the language expresses *only* things the hardware can do efficiently. Not by programmer discipline. Not by convention. By construction — because the type system does not permit anything else.

| Prohibited | Why hardware hates it |
|---|---|
| Hidden copies | Memory bandwidth is finite; unplanned copies consume it invisibly |
| Surprise allocations | Cache misses from heap growth destroy latency predictability |
| Pointer indirection | Pointer chasing defeats the prefetcher; every dereference is a potential cache miss |
| Undeclared timing | Hardware conflicts are invisible until they appear as variance |
| Implicit barriers | Pipeline stalls from barrier instructions are structural overhead |
| GC pauses | Unpredictable stop-the-world violates every real-time contract |

Each of these is not prohibited by a style guide or a linter. It is prohibited by the type system: `Channel<T>` transfers ownership without copying; lifetime types eliminate heap allocation on the hot path; array-based collections make pointer chasing inexpressible; cycle annotations make timing explicit; the absence of shared mutable state makes barriers impossible to need; the two-question memory manager makes GC pauses structurally impossible.

If it compiles, the hardware-correctness is not assumed. It is a theorem.

---

## Designed for AI Authorship

Every other systems language was designed for humans to write by hand. The clock-aware language is designed with the assumption that AI writes the code — and the compiler verifies it. This is not an afterthought or a compatibility claim. It is a design principle that shaped every decision in the model.

The reason current AI code generation fails at systems and embedded code is structural: the correctness constraints are not in the source file. Interrupt priorities, stack sizes, RTOS task budgets, hardware timing relationships, memory ordering semantics — these live in datasheets, reference manuals, and tribal knowledge the AI cannot access. The AI generates plausible code. Plausible is not correct. In systems code, incorrect means dangerous.

The clock-aware model eliminates this problem by design: every constraint that matters is either in the type system, in the `@Timeslice` annotation, or in `system.cap`. There is nothing left outside the source file. The AI generates code; the compiler checks every constraint the AI touched and every constraint it didn't think about. The AI does not need to be correct. It needs to compile.

Why the clock-aware language is different:

| Property | Effect on AI |
|---|---|
| 4 language rules | Fits in any context window; AI holds the complete model |
| Declared timing | AI can verify cycle budget correctness before generating |
| Lifetime types | AI knows exactly where each value lives and when it expires |
| `Channel<T>` | AI knows the complete I/O pattern from the subscription list |
| Exhaustive match | AI cannot silently omit a state; every unhandled case must be explicitly declared as ignored or defaulted — the compiler rejects undeclared intent |
| Hardware-correct by type | AI's output is compiler-validated, not human-validated |

The context window requirement for correct systems code collapses to almost nothing. In a conventional model, the AI needs the hardware reference manual, the RTOS documentation, the platform BSP, the team's internal timing budget spreadsheet, the certification standard, and years of tribal knowledge about what the scheduler actually does under load — hundreds of books of context that no finite window can hold, and that changes between projects. In the clock-aware model, all of that context is in the compiler. The hardware model is `system.cap`. The timing constraints are the ALU cost table. The compliance requirements are the declared profile. The memory model is the tier system. The AI needs to know four rules and the type signatures of the functions it is calling. The compiler holds everything else. A context window of four rules is enough to write correct kernel code.

The feedback loop changes from:

```
generate → flash → observe runtime failure → guess cause → fix → repeat
```

to:

```
generate → compile → read precise local error → fix → compile again
```

No hardware required. No flashing. No guessing. The AI does not need to know the 9,000 pages of hardware documentation. The compiler enforces them. The AI follows the types.

This is the first systems model that is genuinely legible to an AI code generator — not because the AI became smarter, but because the model stopped hiding information from it. The information was always there; it was just never declared.

### The Compiler as the Sole Gatekeeper — No Human Required

The completeness of the compile-time proof has a direct consequence for how code can be produced and reviewed: **an AI can generate and another AI can review, with the compiler as the sole arbiter of correctness. No human needs to be in the loop.**

This is not a claim about AI capability. It is a claim about what the compiler guarantees. Every correctness property that matters in this model is checked at compile time:

| Property | Checked by |
|---|---|
| Timing budget honoured | Compiler (ALU cost table vs declared `@Timeslice`) |
| Memory footprint bounded | Compiler (lifetime types, declared tier, declared size) |
| Error signals handled | Compiler (exhaustive match, error-handler pair at registration) |
| Channel types match | Compiler (subscription types checked at the connection site) |
| No unbounded operations on hot path | Compiler (stdlib functions must have declared budgets) |
| No unsafe memory operations | Compiler (no `unsafe`, no pointer arithmetic, no `new`) |
| No hidden shared state | Compiler (no mechanism to express it) |
| Compliance target met | Compiler (constraint profile from `system.cap`) |

None of these properties require a human to read the code and reason about it. They are either proven or the compilation fails. An AI reviewer's job is therefore not to check correctness — the compiler already did — but to check legibility, naming, and whether the declared structure matches the intended design. That job is well within the scope of current models.

The consequence for kernel development is precise: **no garbage can enter the kernel**. A kernel circuit that compiles is one that the compiler has proven timing-correct, memory-correct, error-handled, and compliance-verified. A kernel circuit that does not compile does not ship. The gatekeeper is not a code reviewer, a security team, or a certification engineer. It is the compiler, running deterministically, producing the same verdict on every machine, every time.

This is the model where AI-generated code becomes safe-to-ship in safety-critical systems — not because we trust the AI, but because we have a compiler that makes trust unnecessary.

---

## Relation to the ADRs

The ADRs in this repository record implementation decisions for a near-term Rust prototype: proc-macro annotations, `system.cap`, `Channel<T>`, `cargo-timeslice`. That prototype is a research vehicle — a way to validate the four rules and the verification model on existing hardware using existing toolchain infrastructure. It is not the language described in this paper.

The channel-based I/O model (ADR-0010) is the first instantiation of the unified I/O abstraction. The unified system configuration (ADR-0005) is the first instantiation of the hardware-model declaration. These concepts carry forward directly into the language.

The Rust implementation path does not. The language described in this paper is designed from scratch, written in itself from Generation 1, with no Rust in the stack.

Each ADR is a step toward understanding. This paper is the language destination.

---

*Part of the clock-aware programming series. See [Paper I: Clock-Aware Programming](01-clock-aware-programming.md) for the core primitive. See [Paper III: The OS and Runtime](03-os-and-runtime.md) for the runtime and OS consequences. See [Paper IV: Hardware Architecture Implications](04-hardware-architecture.md) for the silicon consequences.*
