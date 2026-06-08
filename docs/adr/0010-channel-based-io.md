# ADR-0010: Channel-Based I/O — Hardware Signals as Declared-Timing Channels

**Date:** 2026-06-08  
**Status:** Accepted  
**Supersedes:** ADR-0006 (Poll-Mode Self-Regulating Network Stack)

---

## Context

ADR-0006 described the app partition's network receive path as a poll-mode busy loop checking a ring buffer head pointer, with self-regulation on ring depth. That framing was correct as far as it went — it eliminated the Linux interrupt-driven network stack and gave the compiler visibility into the poll cycle.

But it was still special-case thinking. It described the NIC specifically, using "poll loop" language that implies a mechanism distinct from the rest of the system. As the model matured, a deeper unification emerged: not just the NIC, but every hardware signal source — NIC frames, IRQs, timers, inter-partition messages, sensor inputs, actuator acknowledgements — is the same thing in the clock-aware model. They are all channel writes. The receiver reads from the channel at its next declared cycle. There is no interrupt, no poll loop, no wakeup, no special-case mechanism for any of them.

This ADR documents that unification. The decision is not about the network stack specifically — it is about all hardware I/O.

---

## Decision

**Every hardware signal source is a `Channel<T>`. Hardware writes into the channel at its declared rate. The receiving circuit reads from the channel at its next declared cycle. There is no interrupt delivery, no poll loop, no special-case mechanism. All I/O is structurally identical.**

```rust
// NIC receive — hardware DMA writes NicFrame into the channel
#[timeslice(core = 2, cycle = N, budget_ns = 4)]
fn feed_handler(nic: &Channel<NicFrame>, out: &Channel<BookSnapshot>) {
    if let Some(frame) = nic.read() {
        out.write(parse_and_process(frame));
    }
}

// IRQ — hardware writes IrqSignal into the channel; no ISR, no preemption
#[timeslice(core = 0, cycle = M, budget_ns = 2)]
fn irq_handler(irq: &Channel<IrqSignal>, cfg: &Channel<ConfigUpdate>) {
    if let Some(signal) = irq.read() {
        cfg.write(handle_irq(signal));
    }
}

// Inter-partition boundary — OS writes OsEvent; app reads at declared cycle
#[timeslice(core = 2, cycle = N, budget_ns = 1)]
fn boundary_reader(boundary: &Channel<OsEvent>) -> Option<OsEvent> {
    boundary.read()
}
```

Every signal source — NIC, IRQ, timer, inter-partition ring buffer, sensor — is declared in `system.cap` as a named channel with a declared element type, a declared write rate, and a declared physical address. The proc-macro checker verifies that every `Channel<T>::read()` call in the circuit is within the circuit's declared cycle budget, and that the channel is in the circuit's declared subscription list.

---

## What `Channel<T>` is

`Channel<T>` is a typed, declared-timing, zero-copy conduit between a hardware producer and a software circuit. It is:

- **Not a queue with a lock** — there is no `Mutex`, no `RwLock`, no atomic CAS in the read path.
- **Not an async future with a waker** — there is no executor, no waker registration, no task queue.
- **Not a ring buffer with head/tail polling** — there is no explicit "poll" concept; the circuit reads at its declared cycle and the channel either has data or it does not.
- **Not a blocking call** — `Channel<T>::read()` returns `Option<T>`: `Some(value)` if a frame is present at this cycle, `None` if the channel is empty at this cycle. The circuit continues either way.

It is a direct analogue of an HLS `hls::stream<T>`: a typed conduit with declared throughput, verified by the compiler, with no runtime synchronisation mechanism because the producer and consumer cycles are declared non-overlapping.

### Physical implementation

`Channel<T>` maps to a physically contiguous memory region whose address is declared in `system.cap`. For hardware-sourced channels:

- **NIC frames**: the NIC DMA engine writes directly into the channel buffer at the declared physical address. The circuit reads from that address at its declared cycle. Zero copies — the data moves once, from NIC silicon to the channel buffer, by hardware.
- **IRQ signals**: the interrupt controller writes a typed signal word into the IRQ channel buffer at the declared physical address when the hardware condition fires. No ISR is invoked; no CPU is preempted. The signal sits in the channel until the receiving circuit reads it at its next declared cycle.
- **Timer ticks**: a hardware timer writes a tick signal into a `Channel<TimerTick>` at the declared interval. The circuit that needs timing reads from this channel rather than calling `clock_gettime()` or reading `TSC` directly.
- **Inter-partition boundary**: the OS partition writes `OsEvent` values into a `Channel<OsEvent>` at the declared `os_write_cycle`. The app partition reads at its declared `app_read_cycle`. This is the ring buffer boundary from ADR-0005, expressed as a `Channel<T>`.

In `system.cap`:

```toml
[channel.nic-rx]
type          = "NicFrame"
source        = "hardware"
paddr         = "0x100000000"
size          = "2MB"
write_rate_ns = 100        # worst-case inter-frame gap at line rate

[channel.irq-pcie]
type          = "IrqSignal"
source        = "hardware"
paddr         = "0x100200000"
size          = "4KB"

[channel.boundary]
type          = "OsEvent"
source        = "os_partition"
paddr         = "0x200000000"
size          = "2MB"
write_cycle   = 1000       # OS writes every 1000 cycles
```

---

## Self-regulation is channel-depth-driven, not loop-driven

ADR-0006 described self-regulation as a backoff in the poll loop. The channel model reframes this correctly: the circuit reads its own channel depths at the cycle boundary and adjusts its declared cycle budget for the next iteration. There is no "loop" in the traditional sense — the FSM executes, observes channel depths, updates its budget, and the next execution begins at the next declared cycle.

```rust
#[timeslice(core = 2, cycle = N, budget_ns = 4)]
fn feed_handler(nic: &Channel<NicFrame>, cfg: &Channel<ConfigUpdate>, out: &Channel<BookSnapshot>) {
    // process available frame
    if let Some(frame) = nic.read() {
        out.write(parse_and_process(frame));
    }

    // self-regulation: adjust next cycle budget based on channel depth
    // deep channel → run sooner (budget shrinks toward minimum)
    // empty channel → run later (budget expands toward maximum backoff)
    cfg.write(ConfigUpdate::NextBudget(nic.depth_hint()));
}
```

`nic.depth_hint()` is a non-blocking read of the channel's fill level — how many elements are present. This is a single cache-line read from the channel's metadata region, included in the function's declared cycle budget. The result is written as a `ConfigUpdate` back to the circuit's own configuration channel, which the runtime applies at the next cycle boundary. The backoff is declared and bounded in `system.cap`; the compiler proves the maximum inter-poll interval and the worst-case processing latency.

The critical distinction from ADR-0006's loop model: the circuit does not decide when to run again by sleeping or spinning. It declares its next budget and returns. The runtime schedules the next execution at exactly the declared cycle. There is no busy wait, no OS sleep, no timer interrupt. The circuit is simply not executing between its declared cycle windows — the core is idle in a declared, compiler-known state.

---

## Why polling cannot provide real-time guarantees — and the channel model can

This is not a subtle distinction. It is the reason the channel model exists.

**Polling has a structural real-time problem.** A polling system checks whether a signal has arrived at a periodic interval. Its worst-case response latency is the poll interval — the signal may arrive immediately after a poll, and then wait a full interval before being seen. To tighten the latency bound, you shorten the poll interval. To guarantee a 1 µs response time, you poll at sub-microsecond intervals. But the guarantee that the poll will actually fire at the declared interval is a runtime property: it depends on the CPU being available, not preempted, not delayed by a cache miss in the poll path, not interrupted by a higher-priority event. On a general-purpose OS, none of these are guaranteed. Even on a hard RTOS, the guarantee is probabilistic — it holds unless a higher-priority task runs long, a cache miss occurs, or a hardware event fires at an inopportune moment.

Polling does not provide a real-time guarantee. It provides a best-effort latency bound that holds under nominal conditions. This is why real-time systems engineering is full of worst-case analysis tools, jitter measurements, and statistical latency budgets — because the poll-interval-equals-latency-bound claim is not actually true in the presence of any interference.

**Polling is scheduled — and scheduling is the source of jitter.** This is the deeper problem that the latency-bound argument only partially captures. A poll interval is not enforced by physics — it is enforced by whatever mechanism fires the poll: an OS timer interrupt, an RTOS tick, a hardware timer, a `TSC` spin loop. Every one of these mechanisms is itself subject to jitter. An OS timer fires late if the kernel is handling another interrupt. An RTOS tick fires late if a higher-priority task overruns. A hardware timer fires at its nominal interval but the interrupt delivery to the CPU is delayed by the interrupt controller's arbitration. A `TSC` spin loop is preempted if the OS delivers an interrupt mid-spin. The poller is always at the mercy of whatever schedules it — and whatever schedules it introduces jitter because it is itself a runtime mechanism operating without declared timing.

This creates a jitter chain: the poll fires with jitter J₁ from the scheduling mechanism, the signal processing runs with jitter J₂ from cache and pipeline state at the moment of execution, the total response jitter is J₁ + J₂. Real-time systems engineering spends enormous effort measuring and bounding J₁ — hardware interrupt latency characterisation, RTOS tick jitter analysis, timer resolution tuning. This effort exists entirely because the poll interval is enforced by a scheduled mechanism rather than by the instruction stream itself.

**The channel model has no scheduled poller — and therefore no scheduling jitter.** The circuit does not wait to be scheduled. It executes because the CPU eecutes its next instruction at the next cycle. The timing is enfoxrced by the CPU's instruction execution — the most precise timing mechanism available, running at the clock frequency of the silicon. There is no OS timer, no RTOS tick, no interrupt controller, no spin loop between the declared cycle and the circuit's execution. J₁ is structurally zero. The only jitter source is J₂ — microarchitectural variance in instruction execution — which is the margin declared in the `#[timeslice]` annotation and verified to be within the budget by the proc-macro checker. Total response jitter equals the declared margin. No measurement required; no runtime characterisation; no jitter analysis infrastructure. The margin is a compile-time declaration and the proof that it suffices is the build passing. The circuit does not poll. It executes at its declared cycle — that is its real-time deadline, not an approximation of it. The declared cycle is a compile-time constant verified by the proc-macro against the microarchitecture model. The circuit begins execution at cycle N, processes every frame present in its declared channels within its declared budget, and completes before cycle N + budget_cycles. This is not a hope; it is a compile-time proof. If the instruction mix does not fit within the budget, the build fails.

The maximum response latency for any signal is therefore:

```
max_latency = channel_write_interval + circuit_read_cycle_budget
```

Both terms are compile-time constants declared in `system.cap` and the `#[timeslice]` annotation. The latency bound is not measured at runtime — it is derived at compile time from the declarations. It is a property of the source code, not of the execution environment.

This is the same guarantee an FPGA provides. An FPGA circuit that processes a signal on the next rising clock edge after it arrives has a response latency of exactly one clock period — not "approximately one clock period under nominal conditions", but exactly one period, because the hardware is synchronous and the clock is the global synchronisation primitive. The channel model brings the same structural guarantee to software: the declared cycle is the synchronisation primitive, the compiler verifies it is met, and the response latency is a theorem, not a measurement.

**The distinction from ADR-0006's poll-mode framing:** ADR-0006 described a `while true` loop with a backoff. That is polling — the loop checks the ring buffer, backs off if empty, and loops again. The latency bound depended on the backoff being bounded and the loop running at the declared rate, which were runtime properties. The channel model removes the loop entirely. The circuit executes at its declared cycle. The channel either has data (`Some(frame)`) or it does not (`None`). Either way, the circuit completes within its declared budget and returns. The runtime invokes it again at the next declared cycle. There is no loop, no backoff, no polling. There is only execution at declared times.

In conventional systems programming, three distinct mechanisms exist for hardware signal delivery:

| Mechanism | How it works | Cost |
|---|---|---|
| Interrupt | CPU preempted asynchronously, ISR runs | Context switch, pipeline flush, unknown timing |
| Poll | Software loop checks hardware register | Busy wait or OS sleep/wakeup overhead |
| Message | Software queue, producer/consumer | Lock or atomic overhead, scheduling latency |

These three mechanisms are different runtime implementations of the same underlying intent: deliver a signal from a hardware source to a software consumer. They differ only because the timing of the signal was never declared. Without a declared receive cycle, the system must either interrupt the consumer whenever the signal arrives (interrupt), have the consumer check periodically (poll), or buffer the signal and notify the consumer through a scheduling mechanism (message).

When the receive cycle is declared, all three mechanisms collapse to the same thing: the hardware writes into a typed buffer at the declared address, and the consumer reads from that buffer at its declared cycle. The signal either arrived before the read cycle (it's in the buffer) or it didn't (the buffer is empty, `Option::None`). There is no interrupt because the consumer is already scheduled to read at the declared cycle — preemption adds nothing. There is no poll because the consumer reads exactly once per cycle — looping adds nothing. There is no message queue because the channel buffer is the queue and the declared cycle is the schedule — a separate scheduling mechanism adds nothing.

**The three mechanisms do not replace each other. They become unnecessary simultaneously, as a consequence of declared timing.**

---

## Rationale

### Every previous mechanism was compensating for undeclared receive timing

DPDK eliminates the interrupt-driven stack and gives the CPU direct access to the NIC ring. It does not declare when that ring is accessed — it polls. XDP runs before `sk_buff` allocation but is still interrupt-activated. `io_uring` in poll mode reduces syscall overhead but retains kernel involvement. Onload bypasses the kernel for data but is opaque to the compiler. PREEMPT_RT reduces interrupt latency but does not eliminate it.

Every one of these is a partial solution to the same problem: the software never declared when it would receive hardware signals, so it built increasingly sophisticated machinery to deliver those signals with lower and more predictable latency. `Channel<T>` with declared cycles does not improve that machinery — it removes the premise that the machinery was addressing.

### Zero-copy is a consequence of the channel model, not a separate optimisation

When the NIC DMA writes directly into the `Channel<NicFrame>` buffer and the circuit reads directly from that buffer, there are no copies. This is not an optimisation applied on top of a copy-based model — it is the natural consequence of declaring where the data lives (the physical address in `system.cap`) and who reads it (the subscribing circuit at the declared cycle). No intermediate buffers are needed because the producer and consumer addresses are declared and verified at compile time. The compiler knows the data is in the channel buffer at the declared address; there is nowhere else it needs to go.

### Unified channel model simplifies `system.cap` and the compiler's verification domain

ADR-0005's `system.cap` had a `[boundary]` section specifically for the ring buffer. With the channel model, every I/O source is a `[channel.*]` section with the same schema: type, source, physical address, size, write rate. The compiler's verification domain is uniform: for every channel in `system.cap`, verify that every subscribing circuit reads within its declared budget, and that no circuit reads from a channel not in its subscription list. One schema, one verification rule, all I/O sources.

### The channel subscription list is the complete I/O specification

An application's `system.cap` subscription list is not just a security boundary (ADR for app identity) — it is a complete, machine-readable, compiler-verified specification of every signal the application will ever receive. There are no side channels, no implicit inputs, no signals that arrive outside the declared subscription. The application's behaviour is fully determined by its declared inputs and its declared cycle budgets. This is the property that makes the compliance artefact (ADR-0004) complete: the timing proof covers not just the computation but the full I/O graph.

---

## Consequences

- ADR-0006 is superseded. The "poll-mode network stack" framing is replaced by `Channel<NicFrame>`. The self-regulation mechanism is retained but reframed as depth-driven budget adjustment, not loop backoff.
- `system.cap` gains a `[channel.*]` section replacing the `[boundary]` section. The boundary channel becomes `[channel.boundary]` with `source = "os_partition"`.
- The app partition's main loop is not a loop in the `while true` sense — it is the FSM executing at declared cycle intervals, reading from declared channels, writing to declared channels, and returning. The runtime re-invokes it at the next declared cycle. There is no explicit loop construct in the application code.
- Every hardware signal source requires a physical address declaration in `system.cap`. NIC configuration (via `ethtool`, `devlink`) must direct DMA to the declared address. This remains a manual step in the first implementation.
- `Channel<T>::read()` returns `Option<T>`, not a blocking call. A circuit that needs to wait for data declares a longer cycle budget — the compiler verifies it fits. A circuit that processes whatever is available and moves on declares a short budget. The choice is explicit, visible in the annotation, and verified at compile time.
- The maximum signal-to-processing latency for any channel is `declared_write_interval + declared_read_cycle`, provable at compile time from `system.cap`. This bound is included in the compliance artefact.
