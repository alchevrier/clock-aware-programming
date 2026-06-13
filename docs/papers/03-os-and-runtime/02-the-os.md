## The OS

### The Runtime Is the Kernel

The kernel is a collection of timed functions. Each has a declared core, a declared cycle budget, a declared channel subscription list. The kernel's IRQ handler is a timed function. The kernel's RCU reclaim is a timed function. The kernel's memory management is a timed function. Each writes into typed channels; each reads from typed channels; none preempts any other because their windows are declared non-overlapping.

This is identical to an application function. The only distinction between a kernel function and an application function is the channel subscription list. Kernel functions subscribe to hardware-backed channels (`channel IrqSignal`, `channel TimerTick`). Application functions subscribe to logical channels (`channel NicFrame`, `channel OrderDecision`). In both cases the programmer writes the same channel API — the compiler resolves the physical backing from `system.cap`. Both are compiled with the same proc-macro, verified against the same `system.cap`, sealed into the same signed manifest.

The runtime is not a separate thing that sits between the language and the kernel. The kernel is the runtime. Both are collections of declared-timing circuits, compiled and verified by the same toolchain.

### The OS Is a Collection of Circuits

The operating system is not a special-privileged process. It is a collection of declared-timing circuits, written in the language, compiled by the same toolchain, verified by the same compiler. Apart from the ~500 lines of Assembly stubs that handle boot and hardware initialisation, every line of the OS is a function or a class with a `@Timeslice` annotation, a lifetime type, and a channel subscription list.

Put precisely: **the kernel is a circuit whose driving input is `channel CPUClockTick`.** Every other event in the system — a packet arriving, a value crossing a threshold, a circuit completing its window — is a consequence of that one signal propagating through the dispatch table. There is no other source. There is no other entry point. The clock ticks; the kernel advances; circuits execute; channels carry the results forward. The entire execution model reduces to that chain.

There is no privileged mode at the language level. The distinction between an OS circuit and an application circuit is the channel subscription list: OS circuits subscribe to hardware-backed channels (`channel IrqSignal`, `channel TimerTick`, `channel DmaCompletion`); application circuits subscribe to logical channels (`channel NicFrame`, `channel OrderDecision`). The language is the same. The compiler is the same. The four rules are the same.

**A channel is a wire.** In an electronic circuit, a wire carries every signal that flows between two components — it is the complete record of their interaction. Nothing passes between them except through it. In the clock-aware model, a channel is exactly this: every value that flows from one circuit to another flows through a declared, typed, named channel. There is no other path. There is no shared memory address passed as a pointer, no global variable read by convention, no syscall buffer that two components agree to use. Every interaction between circuits is a channel write and a channel read. The channel is not a message queue. It is the wire. The wire carries everything. The wire is the proof.

This is what makes the model tractable. In an electronic circuit, debugging is possible because every signal is observable — you attach a probe to the wire and read what flows through it. In a conventional software system, debugging is difficult because the interactions between components are not on wires — they are implicit in call stacks, shared memory layouts, and protocol conventions that were never formally specified. The clock-aware model puts every interaction back on a wire, gives it a name, gives it a type, and lets the compiler and the `ObservabilityCircuit` probe it continuously. Any channel can be subscribed to by the observability sub-circuit. Any value that ever flowed through it is in the atom log. The probe is always attached.

**A channel is also a physical allocation decision.** The wire metaphor describes the logical property — one writer, declared readers, typed values, no other path. The backing store is a separate, orthogonal decision made by the compiler from the declared tier and the declared timing. The same channel declaration can be physically realised as:

- A **hardware register** — if the channel element type fits in a machine word, the writer and reader share the same core, and the access pattern is within the same window. No memory bus involved. Zero-cycle transfer.
- An **L1 cache line** — if the channel is `ephemeral` or `task` tier, the writer and reader are on the same core or adjacent cores within the same cache domain, and the declared access tick falls within L1 residency. The compiler pins the channel's ring buffer to a fixed cache line address and guarantees it is not evicted between writer and reader windows.
- An **L2 cache region** — if the channel crosses a core boundary within the same socket and the declared `rtt_ns` is compatible with L2 latency. The compiler inserts a coherency gap of the declared cost into the dispatch table between the writer's window close and the reader's window open.
- A **DRAM-backed ring buffer** — if the channel is `session` or `permanent` tier, or if the declared size exceeds L2 capacity. The compiler pre-allocates the buffer at system start and proves the producer write rate fits the consumer read period without overflow.
- A **network-backed buffer** — if the channel has `backing = tcp` or `backing = rdma` declared in `system.cap`, bridging circuits on separate machines. The channel abstraction is identical from the circuit's perspective; only the declared latency numbers change.

The programmer declares one thing: `channel PriceTable { val element = Price; val tier = task; val size = 1024; ... }`. The compiler resolves the physical backing from the tier, the size, the access pattern, and the machine model. The circuit code reads and writes channel elements — it never addresses memory directly, never specifies cache levels, never touches a DMA descriptor. The channel is the abstraction that makes the wire metaphor physically real: one name, one declaration, one proof obligation — backed by whatever physical medium the compiler determines is correct for the declared timing.

**Every inter-circuit result is a channel write — there is no other mechanism.** This follows directly from the model and is worth stating plainly. A function called *within* a circuit's window returns to its caller in the normal way: a value in a register, consumed immediately by the next instruction, within the same `budget_ticks` window. That return never leaves the window and is never visible to any other circuit. It is `ephemeral` tier — it exists for the duration of the expression and nowhere else.

But any value a circuit produces for another circuit to consume must survive the window boundary. The only thing that survives a window boundary is a channel. There is no return value that crosses windows. There is no shared output buffer that two circuits agree to use. There is no global variable written by one circuit and read by another. The circuit's final act before its window closes is a channel write — that write is the circuit's output, in its entirety. The consuming circuit reads it from the same channel at its own window open.

This means the return type of a circuit, as a whole, is always a channel write site. The circuit's declared output is the set of channels it writes before closing its window. The compiler knows exactly which channels those are — they appear in the port utilisation profile, at declared tick positions within the `budget_ticks` window. There is no implicit output. There is no function return that falls through to the next circuit. The channel is the only interface that exists between windows, which means it is the only interface that exists between circuits, which means it is the only interface that exists in the system. Everything else is intra-window arithmetic.

**A circuit is therefore exactly: channels in, channels out.** Its type signature at the system level is the set of channels it reads and the set of channels it writes. Nothing else. The internal computation — the registers manipulated, the stack frames allocated, the functions called — is invisible to every other circuit and invisible to the runtime. The runtime knows only the window boundary and the channel writes that happen at declared positions within it. This is not an analogy to functional programming or dataflow graphs; it is a direct consequence of the window model. Because no value can leave a circuit except through a channel write, and no value can enter a circuit except through a channel read, a circuit's entire observable behaviour is its channel profile. Two circuits with identical channel profiles are indistinguishable to the system — which is exactly the substitutability property needed for live circuit swap, versioned upgrades, and compile-time composition checks.

**The keyword `channel` disappears from the circuit signature.** Because every parameter a circuit receives is a channel read and every value it produces is a channel write, the compiler can infer the channel from the function signature alone. The programmer writes:

```
circuit RiskCheck {
    fn run(msg: RawMessage) -> Price { ... }
}
```

The compiler sees `msg: RawMessage` — a value crossing a circuit boundary at window open — and allocates a `channel RawMessage` with the tier, size, and access pattern derived from the declared type and the circuit's `@Timeslice`. It sees `-> Price` — a value crossing a circuit boundary at window close — and allocates a `channel Price` on the same basis. The explicit `channel` declarations are needed only when the programmer wants to override the defaults: a non-standard size, a specific tier, a custom access pattern, or a multi-producer topology. In the common case, the signature *is* the channel declaration. The wire is inferred from the connection.

This is why the named `channel` block exists as an escape hatch rather than a requirement. `channel PriceTable { val size = 1024; fn writeAccessPattern(...) = ... }` is the programmer saying: *this channel has structure the compiler cannot infer from the type alone*. A plain `-> Price` in a circuit signature is the programmer saying: *one element, one producer, one consumer, default tier — you figure out the rest*. Both are channels. Only one needs to be written.

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

### Live Circuit Swap — Zero-Downtime Hot Reload

Replacing a running circuit with a new version — deploying a patch, updating a trading algorithm, swapping a driver — follows the same `add_circuit` protocol as any other slot operation, with one additional step: ownership transfer.

The protocol:

1. **Stage the new version.** The new manifest is written to a staging slot in the dispatch table alongside the old circuit's live slot. The runtime pre-allocates the new circuit's full working set from the appropriate tier. This happens in the background — the old circuit continues executing in its declared windows while the new version is being prepared.

2. **Signal the old circuit.** At the next window boundary, the runtime delivers `channel UpdateSignal` to the running circuit — the same mechanism as any other signal. The old circuit finishes its current window. It writes any in-flight `session`-tier state that must survive the swap to a declared transfer channel.

3. **Transfer ownership at the next dispatch boundary.** The runtime atomically replaces the old circuit's dispatch table entry with the new one. The new circuit's channel subscriptions are registered in place of the old circuit's. The transfer channel — if declared — is handed to the new circuit's first window as its initial input. The swap costs exactly one `runtime_overhead_ticks` gap between the last old window and the first new window. No frame is dropped. No packet is lost. No connection is interrupted — the `session`-tier channel backing the TCP connection is owned by the new circuit from the moment of the swap.

4. **Reclaim the old circuit's exclusive state.** Any `task`-tier or `ephemeral`-tier state owned by the old circuit is reclaimed immediately at the swap boundary — it did not survive the swap and was not intended to. Any `session`-tier state explicitly excluded from the transfer channel is released back to the memory tier. The old manifest slot is freed.

```
  tick N     old circuit window    (last execution)
  tick N+1   runtime_overhead      (swap: dispatch table entry replaced atomically)
  tick N+2   new circuit window    (first execution, transfer channel available)
             ── no gap in channel delivery ──
             ── no dropped frames ──
             ── session state transferred ──
```

The key constraint: **the swap boundary is a declared tick, not a runtime decision**. The old circuit declares in its manifest which `session` values must survive a swap — the transfer channel is typed and sized at compile time. A swap that would require transferring more state than the declared transfer channel can hold is a compile error. The programmer knows before deployment whether the hot reload is state-safe. The compiler proves it. The runtime executes the proof.

This is categorically different from a container rolling update or a Kubernetes pod restart. Those involve a process boundary, a new address space, a new socket, and a protocol-level handshake to transfer state. A clock-aware live swap has none of those: same address space, same channel backing, same dispatch table, one tick of overhead.

### Kernel Circuit Updates — The Harder Case

Application circuit swap is clean because the channel profile is the complete interface: two circuits with identical channel profiles are substitutable at the dispatch table boundary by construction. Kernel circuit updates are harder because OS circuits are load-bearing for the runtime itself. `MemoryCircuit` manages the tier allocations every other circuit depends on. `ClockCircuit` produces the `channel Tick` that drives the runtime's own dispatch. `NamespaceCircuit` holds the mapping every `add_circuit` call resolves against. You cannot use the standard swap protocol on a circuit whose operation is a precondition for the swap protocol itself.

The model addresses this through two mechanisms.

**Channel name identity is the interface contract.** `MemoryCircuit` is not identified by a manifest hash or a version number — it is identified by the channels it produces: `channel SlotAck` and `channel RemovalSignal`. Any circuit that produces those channels with the same declared types and declared timing is, to the system, `MemoryCircuit`. A new version with a bug fix differs only in its internal computation — the channels it reads and writes are the same names, same types, same sizes. The compiler enforces this: if the new manifest's channel profile does not match the declared interface of the circuit it is replacing, it is a type error. The interface is the channel set. Version identity is implicit in it.

**Load-bearing circuits are swapped in dependency order.** The runtime maintains a dependency graph among kernel circuits derived from their channel subscriptions — who produces `channel Tick`, who consumes it, who produces `channel SlotAck`, who depends on it. A kernel circuit whose consumers are themselves load-bearing for the swap cannot be swapped first. The runtime resolves the update order: leaf circuits (those with no load-bearing dependents) swap first. Load-bearing circuits swap after all their dependents have been updated and are running against the new interface. The `MemoryCircuit` — the most load-bearing of all — swaps last, in a brief quiescent window during which the other kernel circuits have already transitioned.

The quiescent window for `MemoryCircuit` is short by design: the compiler proves the new `MemoryCircuit` manifest's memory footprint is compatible with the current tier allocation, so the swap does not require rebalancing any existing circuit's memory. If it is not compatible — if the new version requires more `task` tier than is currently reserved — it is a compile error before deployment. The kernel update either passes the compatibility check at compile time and swaps cleanly at runtime, or it fails before the first byte is deployed. There is no "the kernel update failed mid-way" scenario. The compatibility proof either exists or the update is not attempted.

**Kernel evolution is two operations and only two.** A kernel update is either a **swap** — same channel profile, new implementation — or an **add** — new channel profile, new capability. There is no third operation. No kernel patch mechanism. No module reload with side effects. No reboot.

- **Swap**: bug fix, performance improvement, algorithm change in an existing kernel circuit. The channel profile is identical. The compiler checks it. The dependency-ordered swap protocol applies. The system does not notice — same channels, same timing, new code.
- **Add**: new OS capability (a new device driver, a new protocol circuit, a new observability sub-circuit). A new circuit with a new channel profile is slotted alongside the existing kernel circuits via the standard `add_circuit` call. No existing circuit is disturbed. The new circuit starts producing its channels at its next declared window boundary.

Every Linux kernel update that requires a reboot does so because the kernel is a monolith — a single binary where changing one subsystem requires reloading the entire thing, and the new version's data structures are incompatible with the running state. Clock-aware kernel circuits have no shared data structures. Each circuit owns its channels. Each channel is typed and bounded. A new `MemoryCircuit` binary carries its own manifest and its own declared channel profile. The only question the swap protocol asks is: does the new manifest's channel profile match the old one? If yes, the swap proceeds. If no, it does not compile. The kernel is a collection of independent circuits. Updating one does not touch the others.

**There is no reboot. Not as a feature — as a logical consequence.** The reason a conventional OS requires a reboot for kernel updates is that the kernel has no interface boundary between its subsystems. Everything shares the same address space, the same lock hierarchy, the same data structures. Updating one part means replacing the whole thing in a consistent state, and the only way to reach a consistent state is to stop everything, swap the binary, and restart. The reboot is not a technical requirement of updating software. It is a consequence of having no interface boundaries to update across.

Clock-aware kernel circuits have explicit interface boundaries by construction — the channels. It is structurally impossible to accidentally share state across that boundary, because the only thing that crosses it is a typed channel write. Updating `NicCircuit` cannot break `MemoryCircuit` — they share nothing. The swap either matches the channel profile or it is a compile error. There is no inconsistent intermediate state. There is no window during which half the kernel is old and half is new. The swap is atomic at the dispatch table boundary — one tick, then the new circuit's window opens.

This extends to security patches and CVE fixes. A vulnerability in the NIC driver is patched by swapping `NicCircuit` with a fixed version. One tick of overhead. No service interruption. No maintenance window. No "patch Tuesday". No scheduled downtime. No rolling restart across a cluster. The running system does not notice — it sees the same channel profile, the same timing, the same declared subscriptions. The internal code changed. The interface did not. That is all the swap protocol needs to know.

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

### The Network Stack Is a Circuit

The network stack is not a separate subsystem. It is a chain of declared circuits, each consuming a channel from the layer below and producing a channel to the layer above — exactly like any other pipeline in the system. There is no special path for network data. There is no kernel network buffer separate from the tier system. There is no `send()` syscall that copies data into a kernel socket buffer.

**Sending** is a channel write. The application circuit writes its payload to a declared `task`-tier channel. The network stack's next circuit — the transport layer — is subscribed to that channel. At its declared window, it reads the payload, frames it, and writes to the `channel EthernetFrame` that the `NicCircuit` consumes. The `NicCircuit` picks it up at its own window boundary and the DMA engine carries it to the wire. The application circuit never called a syscall. It wrote to a channel. The network stack is downstream in the channel graph.

**Receiving** is a DMA write followed by an `early_fire`. The NIC's DMA engine writes the incoming frame directly into `channel EthernetFrame`'s declared physical address — the same address the `NicCircuit` reads from, the same address that was pre-allocated by the runtime at slot time. The DMA completion asserts the `IrqSignal` on `NicCircuit`'s dispatch table entry. The runtime sees the `early_fire` flag on its next WATCH iteration and promotes `NicCircuit` to the head of the dispatch queue. `NicCircuit` executes at the next window boundary — not at an arbitrary interrupt time, not after a scheduler decision, but at the next declared tick — and delivers the frame up the channel chain to whoever subscribed.

```
  Send path:
  AppCircuit writes → channel Payload (task tier, L1)
                          │
                    TransportCircuit reads at next window
                          │ frames + checksums
                    writes → channel EthernetFrame
                          │
                    NicCircuit reads at next window
                          │ DMA to wire
                    ──────────────────────────────► wire

  Receive path:
  wire → NIC DMA writes → channel EthernetFrame (L1-pinned physical address)
                          │ sets early_fire on NicCircuit
                    runtime sees flag on next WATCH iteration
                          │ promotes NicCircuit
                    NicCircuit executes at next window boundary
                          │ delivers frame up channel chain
                    TransportCircuit → AppCircuit
```

The network stack has no concept of push vs pull from the programmer's perspective — data flows through channels in the direction the channel graph declares. Push and pull are implementation details of the tier model: a `task`-tier channel is available for one window and then reclaimed; a `session`-tier channel persists across windows. TCP connection state is a `session`-tier channel — it lives for the duration of the connection and is reclaimed when the circuit that owns it is removed. There is no separate TCP socket buffer. There is no kernel accept queue. There is a declared channel with a declared tier and a declared owner. The connection is the channel.

### Channel Subscription Inference

The programmer does not need to explicitly subscribe to every channel. Subscription is inferred from how the channel is used:

- **A channel passed as a function parameter** is subscribed by default. The compiler sees the function's declared signature, observes the `channel T` parameter, and registers the subscription in the manifest automatically. No annotation required. If the calling circuit is not in the permitted subscriber list in `system.cap`, the registration is a compile error — but the subscription intent itself does not need to be declared separately from the type.

- **A channel declared as internal state** has its role — publisher or consumer — inferred from usage. If the circuit writes to it, it is a publisher. If the circuit reads from it, it is a consumer. If the circuit does both, it is a state owner — a circuit that both produces and consumes the same channel (e.g. an accumulator or a ring buffer manager). The compiler derives the role from the declared operations, not from an annotation. A mismatch — a circuit declared to publish a channel it only reads — is a compile error.

The result is that channel wiring is a consequence of the type system, not a separate declaration layer. The programmer writes functions that take and return `channel T` values. The compiler builds the full subscription graph from the function signatures and usage patterns. The subscription graph is the routing table. It was always implicit in the code — the compiler makes it explicit and verifiable.

---

