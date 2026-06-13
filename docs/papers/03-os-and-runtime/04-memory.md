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

### Redis, LRU, and `mmap` — Infrastructure for Undeclared Timing

Redis, LRU caches, and memory-mapped files are three different answers to the same question the OS cannot answer at compile time: *which data will be needed, when, and how fast must it be available?* Because the OS has no answer, it compensates at runtime. Each compensation mechanism introduces cost that appears nowhere in the code.

**Redis** is an in-process cache outsourced to a separate process. The reason it exists as a separate process is that the application cannot trust that its working set will be in RAM — or even in page cache — when it is needed. The OS may have evicted it. The kernel may be under memory pressure from another process. The JVM's GC may have promoted the data to a cold generation. Redis solves this by keeping the data in its own process's address space, which the OS is less likely to evict because Redis declares its memory limit explicitly (`maxmemory`). Redis is a work-around for the OS not knowing that this data should be pinned. In the clock-aware model, `val tier = session` is that declaration — the compiler pins the channel's memory before the first window opens, the runtime never evicts it, and no separate process is needed. The network hop, the serialisation, the connection pooling, the eviction policy, the replication configuration — all eliminated. The data is pinned DRAM, present at the declared tier, always at the declared latency.

**LRU** is a runtime eviction policy that approximates what the programmer should have declared. Least-recently-used assumes that access recency predicts future access — a heuristic that is sometimes correct and sometimes catastrophically wrong (a one-time sequential scan evicts the entire working set). The LRU cache exists because the programmer did not declare which data is hot and which is cold. In the clock-aware model, `val tier = task` (L1/L2, pinned for the window) and `val tier = session` (DRAM, resident) *are* that declaration. The compiler pins hot data in L1 for the duration of the window and proves it fits. The runtime pre-warms L1 from the dispatch table lookahead before the window opens. There is no eviction during the window — the compiler proved the footprint fits before emitting the binary. The LRU policy is replaced by a compile-time declaration. The programmer does not guess which data is hot; they declare it, and the compiler verifies it.

**`mmap`** is the most seductive of the three because it looks free. `mmap(file, size)` returns immediately. The cost is paid later, invisibly, one 4KB page at a time: a page fault triggers a kernel entry, a TLB fill, a page table walk, a physical page allocation, a disk read, and a TLB shootdown on other cores. The access that caused the fault may have been a single `data[i]` load — indistinguishable in the code from an L1 hit. The actual cost ranges across five orders of magnitude:

| Access scenario | Cycles at 4 GHz |
|---|---|
| L1 hit (page hot, TLB warm) | 4 |
| L2/L3 hit (page in cache) | 12–40 |
| TLB miss (page in RAM, TLB cold) | ~200–1,000 |
| Page fault (page not in RAM) | ~100,000–10,000,000 |

The code is identical in all four cases. The programmer has no way to know which one will occur at runtime — the OS's page eviction decisions are opaque, heuristic-driven, and vary with system load. `mmap` is therefore a performance lottery disguised as a zero-copy interface.

In the clock-aware model there is no lottery. A `session`-tier channel with `size = N` is pre-allocated in physical DRAM at slot time — before the first instruction of the circuit executes. The memory is always physically resident. Page faults do not exist for `session` or `permanent` tier channels because the pages are never unmapped. TLB shootdowns do not occur because the kernel never reclaims the physical pages. The cost is paid once, at slot time, as a subtraction from the pre-allocated tier pool. From that point, every access to the channel is guaranteed to cost the declared tier latency — no variance, no lottery, no invisible kernel entry. A 10 MB price table in `session` tier costs 200–300 cycles the first time the circuit accesses it after a cold start (L3 miss bringing it to L2), and 4–12 cycles on every subsequent access in the same window (L1/L2 hit, prefetcher-primed from the declared access pattern). That is the complete cost profile. The compiler computed it. The manifest records it. The runtime enforces it.

The deeper point is that Redis, LRU, and `mmap` are not different tools for different problems. They are three instances of the same compensation: a system that did not declare what data would be needed when is forced to guess at runtime, and guess wrong enough often enough that elaborate infrastructure is required to manage the guessing. Declare the data's tier and access window, and the infrastructure has nothing left to do.

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

### CLKIN Is the Ground Truth — Wall Clock Is a Label

CLKIN does not drift from the system's perspective. It is the system's perspective. Every timing proof the compiler produces is in CLKIN ticks. Every window boundary the runtime enforces is a CLKIN tick count. The hardware oscillator that drives CLKIN is the only clock that matters for correctness.

What can drift is the mapping from CLKIN ticks to wall-clock time — the label that says "tick 4,000,000,000 corresponds to 2026-06-13T09:00:00.000000000Z". This label is not used by any circuit for execution. It is used only by circuits that need to stamp events with a human-readable timestamp (the `ObservabilityCircuit`, audit logs, the `@Measure` output). The label is managed by the `ClockCircuit`.

The `ClockCircuit` does one thing: it subscribes to an external time source declared in `system.cap` — a PTP hardware timestamp, a GPS PPS signal, an NTP packet received via the `NicCircuit` — and maintains a declared `permanent`-tier mapping record:

```
// system.cap
clock.wall_source    = PTP            // source of external time reference
clock.sync_period_ms = 100            // how often to re-sync the label
clock.max_drift_ns   = 50             // alarm threshold

channel WallOffset {
    val element = TickToNanosecondMapping
    val tier    = permanent
    val size    = 1
}
```

The `ClockCircuit` reads the external time reference at its declared window, computes the offset between the external timestamp and the current CLKIN tick count, and writes the updated mapping to `channel WallOffset`. Any circuit that needs a wall timestamp reads from `channel WallOffset` and applies the mapping to its own CLKIN tick reading. The mapping update is a channel write — declared, typed, observable — not a hidden adjustment to a global variable.

The internal timing proofs are entirely unaffected by this. The compiler proved that `parsePrice` runs for 200 ticks. It still runs for 200 ticks regardless of what the PTP source says about UTC. The only thing that changes when the wall offset is updated is what timestamp the `ObservabilityCircuit` prints next to that 200-tick window in the atom log. Correctness is in ticks. Observability is in nanoseconds. They are separate concerns, managed separately, and the separation is structurally enforced by the type system.

If the external time source becomes unavailable — PTP link down, GPS antenna removed — the `ClockCircuit` signals `channel ClockSyncLost`. Circuits that declared a `maxStaleness` on their timestamp reads receive a `StalenessViolation`. The system continues executing at full correctness in ticks; only the wall-time label on events becomes stale. The declared staleness handler decides whether to continue with a stale label or to degrade gracefully. At no point does clock synchronisation loss affect the timing proofs or the dispatch table.

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

**The runtime's own overhead is included in the footprint at compile time.** The runtime accesses its own data structures on every dispatch loop iteration: the dispatch table entry for the current window, the manifest record for the upcoming circuit, the `early_fire` flag array, the observability delta write buffer. These are memory accesses. Their sizes are known at compile time — the runtime is compiled by the same toolchain against the same `cpu_model`. The compiler measures the runtime's own L1 footprint and includes it in the capacity check alongside the circuit data:

```
  Total L1 footprint check (hot path):

  Circuit working set:     1628B   (channel data across hot-path chain)
  Runtime dispatch data:    256B   (dispatch table window + manifest entry +
                                    early_fire flags + observability buffer)
  ─────────────────────────────
  Total:                   1884B
  L1 capacity:            32768B   (from system.cap)
  Margin:                 30884B   ✓
  hot_path_dram_loads:        0    (proved)
```

A proof that omits the runtime's footprint is not a proof — it is an optimistic estimate that fails the moment the runtime's own accesses evict a circuit's cache line. The compiler includes both because the runtime is not a privileged external entity that sits outside the memory model. It is code, compiled to instructions, accessing memory, counted in ticks and bytes the same as every circuit it runs. Its overhead was already accounted in the instruction budget (Step 2 above). Its memory footprint is accounted here. There is no hidden cost anywhere in the system.

**When the hot path spills, the runtime can move itself to a different core.** If the combined footprint — circuit working set plus runtime dispatch data — exceeds L1 capacity on the hot core, the compiler reports the spill and the available remedies in order:

1. **Reduce the circuit's working set** — split the circuit, narrow a channel element type, demote a value to L2.
2. **Assign the runtime's dispatch management to a dedicated cold core.** The hot core's L1 is then entirely reserved for circuit data. The runtime's dispatch loop executes on its own cold core — with its own L1, its own `cpu_model` (cold cores run at `Efficiency` frequency, sufficient for counter reads and flag scans), its own declared `@Timeslice` window. The hot core receives its EXECUTE trigger via a single cross-core signal from the runtime core — one cache-line-sized message, coherent via ACP, costing the declared cross-core latency gap. The hot core's L1 sees only circuit data. The runtime's accesses never appear in that cache.

This is a natural application of the hot/cold core split already in `system.cap`. The runtime declares itself on a cold core the same way any other circuit is placed — `@Timeslice(core = 0, ...)` for the runtime dispatch loop, `@Timeslice(core = 2, ...)` for the hot-path circuits. The compiler re-runs the L1 footprint check with the runtime footprint removed from the hot core's capacity equation. If the hot-path working set now fits in L1 alone, `hot_path_dram_loads: 0` is restored. The proof holds again, at the cost of one cold core dedicated to runtime management — a cheap core, running at low frequency, doing a small fixed amount of work per tick.

**The canonical topology: one OS core, N application cores.** The logical conclusion of this reasoning is a fixed core assignment declared in `system.cap`:

```
system.cores.os  = [0]          // one dedicated OS core
system.cores.app = [1, 2, 3, 4] // all remaining cores: application circuits only
```

Core 0 runs everything the OS needs: `ClockCircuit`, `MemoryCircuit`, `NicCircuit`, `ObservabilityCircuit`, the runtime dispatch loop, `early_fire` flag scanning, power mode management, prefetch issuance, ACP EXECUTE signals to application cores. Its L1 holds all runtime state. It runs at `Efficiency` frequency — counter reads, flag scans, and management signals do not require 4 GHz.

The OS core's L1 is itself subject to the same compile-time footprint discipline as application cores. The compiler identifies which kernel elements are on the critical path — accessed on every WATCH iteration — and proves they are always L1-resident:

| Critical element | Size | Access frequency | Must be L1-resident |
|---|---|---|---|
| Dispatch table hot window (next N entries) | N × 32B | Every WATCH iteration | Yes |
| `early_fire` flag array | 1 bit per circuit slot | Every WATCH iteration | Yes |
| Manifest records for next N circuits | N × 128B | Per PLAN transition | Yes |
| ACP EXECUTE signal buffer | 64B (one cache line) | Per window boundary | Yes |
| Observability delta write buffer | 256B | Per EVALUATE transition | Yes |

The compiler computes the total footprint of these structures — determined by the number of circuit slots N declared in `system.cap` — and checks it against the OS core's L1 capacity. If it fits, every WATCH iteration reads only from L1. No L2 access, no latency penalty, no degradation in `early_fire` scan speed. If it does not fit, the compiler reports the overflow and the remedy: reduce N (fewer simultaneous circuit slots), widen the OS core's declared L1 (a `system.cap` hardware declaration), or split the dispatch table across two OS cores.

The `early_fire` flag array is the most latency-sensitive element. It is scanned on every WATCH iteration and its result determines IRQ response time. A flag array that spills to L2 adds L2 latency — typically 4–12 ticks — to every scan. On a 4 GHz core that is 1–3 ns added to the worst-case IRQ response bound, which the compiler reports as a change to the proved `irq_response_ticks` constant in the manifest. Nothing is hidden. The footprint is declared. The latency consequence is computed. The programmer sees both.

**Every OS data structure is a fixed-size array.** This is not a design choice — it is an inevitable consequence of the declaration model. Because every resource in the system is declared in `system.cap` with a known count and a known element size, every OS data structure has a compile-time constant size. There are no dynamic allocations in the OS. There are no linked lists with unknown length. There are no hash maps that resize. There are no growable buffers.

| OS data structure | Size derivation | Element size |
|---|---|---|
| Dispatch table | `max_circuit_slots × window_entry_size` | 32B |
| `early_fire` flag array | `max_circuit_slots ÷ 8` (1 bit per slot) | — |
| Manifest registry | `max_circuit_slots × manifest_record_size` | 128B |
| Observability delta buffer | `declared in system.cap` | 8B (one Tick) |
| ACP EXECUTE signal buffer | `app_core_count × cache_line_size` | 64B |
| Key ring | `max_keys × key_size` | 32B |
| Core power state table | `core_count × 1B` | 1B |

Every size in that table is a product of two compile-time constants from `system.cap`. The compiler knows the total byte count of the entire OS before emitting a single instruction. The L1 footprint check is integer arithmetic over known constants. The result is either "fits" or "does not fit by N bytes" — not a measurement, not a profile, not a heuristic. A theorem over declarations.

**`max_circuit_slots` is a ceiling, not an expectation.** The dispatch table is sized for the maximum number of concurrently active circuits declared in `system.cap`. But circuits are not independent — they are dependency chains. A circuit is absent from the dispatch table when its input channel has no data. Because all data flow is through declared channels, the absence of data at the head of a chain cascades: if `NicCircuit` has no frame to deliver, `parsePrice` has no Price to produce, `updateBook` has no OrderBook to update, `emitQuote` has no Quote to emit. Four slots clear simultaneously from one empty channel.

The effective dispatch table occupancy at any moment is bounded by the number of currently active chains multiplied by their average length — not by `max_circuit_slots`. On a quiet system with no incoming packets, the table contains only the kernel circuits (`ClockCircuit`, `ObservabilityCircuit`, and whatever background work is declared) — a handful of entries regardless of how large `max_circuit_slots` is. On a burst, the full active chain appears. Between bursts, it collapses back to the kernel floor.

```
  max_circuit_slots = 64   (declared in system.cap)

  Quiet system (no packets):
    active: ClockCircuit, ObservabilityCircuit = 2 slots occupied

  Single packet burst (one chain fires):
    active: NicCircuit → parsePrice → updateBook → emitQuote
            + kernel floor = 6 slots occupied

  Effective peak occupancy for this workload: 6 / 64 = ~9%
```

The `max_circuit_slots` declaration therefore does not bound performance — it bounds the worst-case OS data structure size and the worst-case L1 footprint check. The actual runtime occupancy is determined by the arrival rate of data at chain heads, which is a property of the workload, not of the system configuration. The system is sized for the declared maximum; it operates at the workload's natural occupancy, which is always less.

The epoch period of the chain head determines how much time the dispatch table has between bursts. Two workloads at opposite ends of the spectrum:

```
  HFT packet processing (10 Gbps NIC):
    Frame period:    1.2 µs  =     4,800 ticks at 4 GHz
    Chain cost:      ~200 ticks (parsePrice → updateBook → emitQuote)
    Empty ticks:     4,600 ticks per epoch
    Occupancy:       ~4%

  Video game at 60 FPS (frame-capped):
    Frame period:    16.67 ms = 66,680,000 ticks at 4 GHz
    Chain cost:      ~8 ms    = 32,000,000 ticks (physics + AI + render)
    Empty ticks:     34,680,000 ticks per frame
    Occupancy:       ~48%
```

The video game case is instructive precisely because it is the hardest case for occupancy — yet even there the dispatch table is empty for more ticks than it is occupied. The frame cap is a declared epoch period: `@Timeslice(period = "16.67ms")` on the frame-head circuit. Every tick of that 16.67 ms is accounted for at compile time. The runtime knows that for the next 34 million ticks after the frame pipeline completes, no frame-chain circuit can fire. It uses that window to run background circuits — asset streaming, audio mixing, network state updates — at their own declared periods, filling the idle ticks with declared work rather than spinning. The dispatch table is never full because the chain head's epoch period guarantees a known-empty interval after every burst, and the compiler schedules background work into that interval explicitly. The frame cap is not a performance limit. It is a scheduling declaration that gives the compiler a provably large idle window to work with.

**A single core with a single-threaded dispatch table delivers stellar performance.** In a conventional OS, single-core means time-slicing: threads compete for the CPU, each paying context-switch overhead at every quantum boundary — register save and restore, cache flush, TLB invalidation, scheduler decision, cache re-warm. The overhead is not optional. It is the cost of the scheduler not knowing what runs next.

In the clock-aware model there is no context switch. There is only the dispatch table advancing one entry at a time. On a single core the pipeline stages run back-to-back: `parsePrice` completes, its output registers are forwarded directly to `updateBook`'s first instruction, `updateBook` completes, its L1-pinned output is read by `emitQuote`. The inter-window cost is `runtime_overhead_ticks` — the counter read and dispatch table advance, a handful of instructions, the same on one core or sixteen. No register save. No cache flush. No scheduler decision. The "threads" are consecutive dispatch table entries whose register state is forwarded rather than saved.

```
  Conventional OS, single core, 3 threads:

  thread A runs ──► context switch (save A, load B) ──► thread B runs
                    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                    ~1000–5000 ticks per switch
                    cache cold, TLB partially invalidated

  Clock-aware, single core, 3 circuit windows:

  parsePrice ──► runtime_overhead (8 ticks) ──► updateBook ──► (8 ticks) ──► emitQuote
                 ^^^^^^^^^^^^^^^^^
                 counter read + dispatch table advance only
                 registers forwarded, L1 hot, no save/restore
```

The single-core clock-aware system is faster than a multi-threaded conventional system for the same pipeline because it eliminates every source of inter-window overhead that multi-threading was supposed to hide. Multi-threading in a conventional OS exists to keep the CPU busy while one thread waits for I/O or memory. In the clock-aware model, the circuit is simply absent when its channel has no data — the core moves to the next dispatch table entry with no overhead. There is nothing to hide, nothing to overlap, nothing to context-switch around. The single core executes exactly what the compiler declared, in order, with the overhead the compiler counted, and nothing else.

**The root cause multi-threading hides is the memory stall — and the clock-aware model eliminates the stall itself.** A conventional thread issues a load, misses L1, misses L2, and waits 100–300 ticks for DRAM. The scheduler switches to another thread to keep the CPU busy during that wait. The thread switch is not a solution — it is a compensation for a stall that should not have happened. The stall happened because the data was not in L1. The data was not in L1 because nobody knew the thread would need it until the instruction that needed it executed.

The clock-aware model knows every memory access the next circuit will make before that circuit's window opens — from the manifest and the channel graph. It issues exact prefetches during the preceding WATCH ticks. By the time the circuit's first instruction executes, the data is already in L1. The stall never occurs. The compensation mechanism — thread switching — is therefore structurally unnecessary. There is no stall to hide.

The OS-on-one-core topology reinforces this. The runtime's management accesses (dispatch table, flag array, manifest records) live exclusively in the OS core's L1. They never appear on an application core. An application core's L1 is entirely circuit data — the exact working set the compiler proved the circuit needs, pre-positioned by the runtime's prefetch, verified to fit by the compile-time footprint check. Every load the circuit issues hits L1. Every load was known in advance. Every load was already there.

```
  Conventional multi-threading:
  thread issues load → L1 miss → L2 miss → DRAM wait (100–300 ticks)
                                              ↓
                                       scheduler switches thread
                                       (compensates for the stall)

  Clock-aware single core:
  WATCH: runtime reads channel graph → issues prefetch for next circuit's data
  EXECUTE: circuit's first load hits L1 (data already there)
           stall: 0 ticks
           thread switch: structurally impossible — there is no thread
```

This is why single-core clock-aware outperforms multi-threaded conventional for the same pipeline. Not because it runs faster instructions. Because it never stalls. A CPU that never stalls on memory has no need for thread switching to hide those stalls. The performance comes from eliminating the problem, not from compensating for it faster.

### What the Programmer Declares. What the Compiler Owns.

Every mechanism discussed in this paper — dispatch tables, epoch periods, pipeline fill costs, register forwarding, speculative regions, L1 footprint proofs, prefetch issuance, power mode transitions, early_fire promotion, cross-core coherency gaps — is the compiler's responsibility, not the programmer's.

The programmer declares three things:

1. **What the data is** — the channel element type, concrete and named.
2. **How long it lives** — the lifetime tier: `register`, `ephemeral`, `task`, `session`, `permanent`, or `cold`.
3. **When the circuit runs** — the `@Timeslice` annotation: which core, what period, what budget.

Everything else follows from those declarations. Threading does not exist as a concept the programmer manages — circuits execute in their declared windows and the compiler proves they do not conflict. Concurrency does not exist as a problem the programmer solves — data flow is through channels with declared ordering, and the compiler proves every handoff arrives before it is read. Pipelining is not something the programmer constructs — the compiler arranges consecutive windows on the same core with register forwarding between them. Cache management is not something the programmer controls — the lifetime tier is the cache tier, and the runtime pre-positions every value before its window opens.

The programmer never thinks about any of this unless she has a **critical path** that is approaching its declared budget. In that case the compiler tells her exactly why: which instruction sequence is the bottleneck, how many ticks it costs, how much margin remains, and which remedies are available (narrow a type, split a circuit, promote to a shorter-period window, move to a core with a larger L1). The information is exact. The decision is hers. The implementation of the decision — register reuse, speculative pipeline, cross-core placement — is again the compiler's.

```
  Programmer's cognitive model:

  "What is this data?"          → val element = Price
  "How long does it live?"      → val tier    = task
  "When does this circuit run?" → @Timeslice(core = 2, period = "1.2us", budget = "800ns")

  Compiler's cognitive model (invisible to programmer unless budget exceeded):

  pipeline fill cost    register forwarding    speculative regions
  L1 footprint check    prefetch scheduling    cross-core coherency gap
  dispatch table slot   early_fire flag bit    power mode transition
  epoch period ordering handoff proof          DRAM load count proof
```

The conventional alternative requires the programmer to manage all of the right column explicitly and correctly, under time pressure, with tools that measure rather than prove, on a system that is simultaneously running an OS whose timing is unknown. The clock-aware model moves the entire right column into the compiler. The programmer writes declarations. The compiler writes proofs. The hardware executes them.

### The Lie of Imperative Concurrency

Imperative programming presents concurrency as the natural state of computation: spawn threads, let them run in parallel, share memory freely. This is not what hardware does. Hardware executes one instruction per cycle per core. Memory bus arbitration is sequential. Cache coherency protocol is sequential. Every access to shared state resolves to a serialised sequence of loads and stores. The parallelism is an abstraction; the sequentiality is the physical truth.

The lock reveals this. When a programmer places a lock around shared state, she is not adding sequentiality — she is admitting that it was always there. She is acknowledging that two threads cannot safely touch the same memory at the same time, that the hardware will serialise them anyway at the cache coherency level if she does not, and that the program's correctness depends on a specific serialisation order. The lock is not a tool for managing concurrency. It is an apology for having pretended that concurrency existed in the first place.

The deeper problem is that locks prove nothing at compile time. The programmer writes a lock, the compiler emits the fence instructions, and the program runs — and at runtime, two threads contend for the same lock, one waits, the CPU stalls, and the scheduler intervenes. The intervention has unknown duration. The duration was unknown because the concurrency model that made the lock necessary also made the waiting time unprovable. The lock was the solution to a problem the threading model created.

**In reality, everything is event-driven and sequential.** A packet arrives — an event. A price update is computed — an event. A quote is emitted — an event. Each event has a defined cause, a defined output, and a defined duration. Nothing that happens at the silicon level is truly parallel in the way the threading model implies. The threading model imposed an approximation of parallelism on top of sequential hardware, then introduced locks to repair the cases where the approximation broke.

The clock-aware model starts from the truth. Every circuit is event-driven: it executes when its channel has data, not when a scheduler decides. Every circuit is sequential: it has a declared window, it runs to completion, and no other circuit on the same core touches the same data during that window — proved by the compiler, not asserted by a lock. The channel graph is the causal graph of the entire system, declared statically, verified structurally. There is no shared mutable state between circuits — only channels, which are single-writer, and which transfer ownership of a value at a declared moment.

```
  Imperative concurrency model:
  threads share memory → require locks → locks serialise → admit sequentiality
  lock contention → scheduler intervention → unknown wait → unknown latency

  Clock-aware model:
  circuits own their data → channels transfer ownership → no sharing
  ordering proved at compile time → no lock needed → no contention possible
  sequentiality is declared, not recovered
```

A lock is a runtime mechanism for enforcing what the clock-aware model enforces at compile time: that at any given moment, exactly one piece of code has access to a given value. The difference is that the clock-aware compiler proves this statically, for every value in the system, before a single instruction is emitted. The programmer never writes a lock because the compiler has already guaranteed that no two circuits can conflict — by construction, from the declarations. The sequentiality that locks were invented to enforce is the model, not an afterthought.

### Why OOP Made This Worse — and Why Message-Passing Almost Solved It

Object-oriented programming deepened the problem by hiding state mutation behind method calls. An object holds state. A method mutates it. Another method reads it. The sequence of calls across object boundaries determines the sequence of state changes — but that sequence is never declared. It is implicit in the call graph, reconstructed at runtime by whoever happens to call what in whatever order. The compiler sees individual method bodies. It does not see the global mutation sequence. Nobody sees it, because it was never written down.

The result is that the happens-before relationship between any two state accesses in a large OOP codebase is unknowable by inspection. The programmer cannot look at two reads and determine, statically, which write each will observe. The compiler cannot determine it either. The language specification therefore introduces a **memory model**: a set of rules defining which orderings the compiler is permitted to assume, which it must preserve, and which it may reorder for optimisation. The programmer must reason about the memory model to write correct concurrent code. The memory model exists entirely because the sequence of mutations was never declared.

The irony is that the same programmer, when writing a TCP server or a Kafka consumer, finds the ordering problem completely natural. A TCP stream delivers bytes in order. A Kafka partition delivers messages in order. Causality is explicit: the sender wrote the message before the receiver read it — by construction, from the protocol. The happens-before relationship is not inferred from a memory model; it is guaranteed by the channel. The programmer does not think about orderings on the wire because the wire declares them.

The clock-aware model brings the channel inside the program. Every value moves through a declared channel. A channel has exactly one writer and one or more readers. The writer's window ends before the reader's window begins — the coherency gap is a compile-time constant, proved by Pass 1 of the compiler. The happens-before graph is the channel graph. It was declared by the programmer when she named the channel. The compiler does not infer it. It verifies it.

**The consequence is that no memory model is needed.** A memory model exists to constrain what an optimising compiler may reorder across thread boundaries. In the clock-aware model there are no thread boundaries — only window boundaries, which are declared. The compiler emits instructions in declaration order within each window. Between windows on the same core, register forwarding transfers values directly through the pipeline. There is no reordering ambiguity because there is no concurrency to create ambiguity. The instructions are ordered. The channels are ordered. The cores are in-order processors executing declared sequences.

```
  OOP mutation sequence:
  methodA() → object.field = x        (write)
  methodB() → y = object.field        (read)
  Question: does B see A's write?
  Answer: depends on thread scheduling, memory model, fence placement
          → unknowable by inspection, requires runtime memory model

  Clock-aware channel sequence:
  circuit A window [tick 0–200]  → writes channel C
  circuit B window [tick 210–400] → reads channel C
  coherency gap: 10 ticks (compile-time constant, Pass 1 verified)
  Question: does B see A's write?
  Answer: yes, proved. By construction. Before the first instruction was emitted.
```

Dispatching all circuits in a chain to the same core on an in-order processor is the final simplification. An in-order processor executes instructions in the order they are issued. Registers forwarded between windows are already in the pipeline. L1 loads hit in a fixed number of ticks counted by `llvm-mca`. There are no out-of-order reorderings to reason about, no store buffer drains to fence, no coherency protocol messages to wait for. Correctness is a consequence of declaration order and instruction order being the same thing. The memory model was the patch applied to a system where those two things had come apart. Reattach them and the patch is unnecessary.

### The UML Sequence Diagram Is Finally Correct

A UML sequence diagram shows actors as vertical lifelines and interactions as horizontal arrows. `NicCircuit` calls `parsePrice`. `parsePrice` calls `updateBook`. `updateBook` calls `emitQuote`. The diagram implies a definite sequence: these calls happen in this order, each completing before the next begins, the data flowing cleanly from left to right.

In a conventional program this diagram is aspirational, not accurate. The programmer drew what she intended. What actually executes is different. Between any two arrows on the diagram the scheduler may preempt the thread and not resume it for 1–5 ms. A lock acquisition may put the thread to sleep for an unbounded duration while another thread holds it. A page fault may stall the thread while the kernel fetches a page from DRAM. An SMI fired by firmware may steal the CPU with no record left in any log. The sequence diagram cannot represent any of these events because none of them were declared — they are consequences of a runtime environment that the diagram has no notation for.

The diagram is a fiction. It describes the intended interaction topology, not the actual execution sequence. Software architects have accepted this quietly for decades: the diagram is a communication tool, not a specification. It communicates structure. It does not specify timing, ordering guarantees, or correctness conditions under concurrent execution.

**In the clock-aware model, the sequence diagram is the specification.** The circuit graph is the sequence diagram. Every arrow in the diagram corresponds to a declared channel with a proved coherency gap. Every lifeline corresponds to a circuit with a declared window measured in exact CLKIN ticks. Every interaction on the diagram executes in exactly the order shown, in exactly the window declared, with no unrepresented events between the arrows.

```
  UML sequence diagram in conventional program:

  NicCircuit        parsePrice        updateBook        emitQuote
      |                 |                 |                 |
      |──────frame──────▶                 |                 |
      |                 │  [scheduler may preempt here]     |
      |                 │  [lock may sleep here]            |
      |                 │  [SMI may fire here]              |
      |                 ├──────price──────▶                 |
      |                 |                 │  [page fault?]  |
      |                 |                 ├──────quote──────▶
      |                 |                 |                 |

  The arrows are correct. Everything between them is unknown.

  UML sequence diagram in clock-aware program:

  NicCircuit        parsePrice        updateBook        emitQuote
  [tick 0–200]     [tick 210–400]   [tick 410–600]   [tick 610–800]
      |                 |                 |                 |
      |──channel EthernetFrame──▶        |                 |
      |    (gap: 10 ticks, proved)        |                 |
      |                 |──channel Price──▶                 |
      |                 |    (gap: 10 ticks, proved)        |
      |                 |                 |──channel Quote──▶
      |                 |                 |  (gap: 10 ticks, proved)
      |                 |                 |                 |

  The arrows are correct. Everything between them is also correct.
  There is nothing between them.
```

The diagram is no longer aspirational. It is the compiled output. The compiler's Pass 1 walks the channel graph, verifies every coherency gap, and rejects the program if any arrow in the diagram cannot be proved to carry data before the next lifeline's window opens. What the architect drew is what the hardware executes. The notation that spent decades as a communication fiction becomes a formal verification artefact.

The further consequence is direct: the architect can provide the diagram to an AI code generator and the program can be generated from it mechanically. This is not possible in the conventional model because the diagram does not contain enough information to generate correct concurrent code — it omits the scheduler, the locks, the memory model, the thread lifecycle, the error handling for contention. An AI generating code from a conventional sequence diagram is guessing at all of those. The resulting code may match the diagram's topology while being subtly wrong in ways that only manifest under load, in production, once.

In the clock-aware model the diagram contains all the information. Each lifeline is a `circuit` declaration with a name and a `@Timeslice`. Each arrow is a `channel` declaration with an element type and a lifetime tier. The causal ordering shown in the diagram maps directly to the channel graph that the compiler's Pass 1 verifies. An AI generating clock-aware code from a sequence diagram is not guessing — it is transcribing. The diagram is already the program; the AI is performing a mechanical translation from one notation to another. The compiler then proves the translation was correct or reports the exact line where it was not.

This closes the loop between architecture and implementation that software engineering has pursued since the first CASE tools in the 1980s. Those tools were not wrong. Their premise was correct: the diagram should be the program. The reason they failed is that the software model underneath them was wrong. The diagram described a clean sequential interaction topology. The software model beneath it was imperative, concurrent, lock-based, scheduler-dependent — a model where the same diagram could correspond to infinitely many different runtime behaviours depending on thread interleaving, lock contention, and scheduler policy. No code generator could bridge that gap because the gap was not a tooling problem. It was a model problem. The tools were trying to generate correct concurrent code from a diagram that was structurally incapable of specifying concurrency correctly. They couldn't succeed because the target language made success impossible.

Remove the wrong model. Replace it with declared channels and proved windows. The diagram becomes the program. The tools were right all along.

Cores 1–N run application circuits exclusively. Their L1 holds only circuit data — declared channel buffers, task-tier values, the working set the compiler proved fits. No runtime data structure ever touches their cache. No dispatch loop instruction ever runs on them. They receive a single ACP cache-line EXECUTE signal from core 0 at the start of each window, execute their declared instruction sequence, and return to `STANDBYWFI`. Their entire L1 budget belongs to the application.

```
  core 0 (OS core, Efficiency)          cores 1–N (app cores, Performance)
  ┌────────────────────────────┐        ┌──────────────────────────────┐
  │  WATCH loop (spinning)     │        │  STANDBYWFI                  │
  │  dispatch table            │        │  (waiting for EXECUTE signal) │
  │  early_fire flag scan      │        │                              │
  │  ClockCircuit              │        │                              │
  │  MemoryCircuit             │──ACP──►│  EXECUTE circuit window      │
  │  NicCircuit                │signal  │  (pure circuit data in L1)   │
  │  ObservabilityCircuit      │        │                              │
  │  prefetch issuance         │◄─ACP──│  window close delta write    │
  │  power mode management     │        │  back to observability buf   │
  └────────────────────────────┘        └──────────────────────────────┘
```

This topology eliminates the last source of interference between the OS and application circuits. It is not a heuristic — it is a structural separation declared in `system.cap` and enforced by the compiler. An application circuit assigned to an OS core is a compile error. A runtime data structure allocated in an application core's L1 region is a compile error. The boundary is hard, proved, and permanent.

The stalling risk disappears entirely. A stall on an application core can only come from that core's own instruction sequence — and the compiler already proved the instruction count fits in the declared budget. There is no OS noise, no runtime interrupt, no management access that can land on that core. The application core is an execution unit. The OS core is the controller. They communicate through one declared channel: the ACP EXECUTE signal. Everything else is isolated by construction.

**This is categorically different from DPDK.** DPDK uses the same one-master-lcore, N-worker topology and achieves kernel-bypass I/O by polling the NIC directly from userspace. It is the closest existing approximation of this model — and it falls short in one irreducible way: **DPDK runs on top of the Linux scheduler**. The Linux scheduler still owns the CPU. A DPDK worker thread pinned to a core with `isolcpus` and `SCHED_FIFO` is still a Linux thread. The kernel can still preempt it — a high-priority kernel thread, a softirq, an RCU callback, a memory management operation. An SMI fired by firmware is completely invisible to the OS and to DPDK; it steals CPU time with no record and no bound. `RDTSC` in a DPDK worker measures wall time including all of these; it cannot separate DPDK execution time from stolen time. The worker has no knowledge of whether it was actually running for any given tick.

The clock-aware OS core is not on top of a scheduler. It is the scheduler. No kernel owns the tick above it. No SMI fires outside its declared windows — SMIs are firmware interrupts that require the CPU to be in normal execution mode; a system that has replaced the conventional OS with the clock-aware runtime and verified all firmware interactions through `system.cap` has no path for an unaccounted SMI to reach an application core. The application core receives its EXECUTE signal from the OS core, runs its declared window, and returns to `STANDBYWFI`. The only code that executes on that core is the code the compiler counted. Every tick is attributed. Every tick was declared.

### The Runtime Adapts — AI-Regulated OS

The atom stream, the ML execution planner, and the clock model assignment together form a system that adapts in real time to the actual workload — not by guessing, not by sampling, but by reading a hardware-sourced proof stream and acting on it within the constraints of the compiler's theorems.

The logical conclusion of this architecture is that the OS itself can be regulated by an AI circuit: a declared `AdaptationCircuit` that subscribes to `channel AtomStream`, runs inference over the execution history, and emits `channel RuntimeDirective` — window size adjustments, core placement recommendations, clock model pre-assignments — that the runtime applies at the next dispatch boundary.

This is not an AI that controls the OS in an unconstrained way. It is a circuit like any other circuit: declared timing, declared channels, compiler-verified. Its directives are bounded by the compiler's hard floor. It cannot make the system unsafe — the compiler already proved safety. It can only find and exploit slack that the static proof left on the table, because the static proof is conservative by necessity and the AI has the runtime's live data.

The result is an OS that is simultaneously formally verified (by the compiler) and continuously optimised (by the AI circuit) — not a trade-off between the two, but both at once, at different layers.

---

