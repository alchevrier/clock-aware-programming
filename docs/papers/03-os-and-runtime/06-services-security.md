## Services and Security

### Microservices Are Circuits with Declared Handoffs

This model scales directly to what is conventionally called a microservice architecture — except without the network, without the serialisation, without the service mesh, without the latency. And it scales further — across physical machines — by the same mechanism.

**A TCP connection is a `session`-tier channel whose backing is network instead of DRAM.** Within a machine, two circuits hand off a `session`-tier value by writing to a DRAM address at a declared tick and reading from it at the next declared tick. Across machines, the mechanism is identical except that the physical backing is a network link instead of a memory bus, and the coherency gap is the declared round-trip latency of that link instead of the L3 coherency latency of a shared cache domain. The programmer declares the cross-machine channel in `system.cap`:

```
// system.cap
channels.OrderStream.backing  = tcp
channels.OrderStream.endpoint = "10.0.0.2:9000"
channels.OrderStream.rtt_ns   = 50_000        // declared RTT + jitter margin

channel OrderStream {
    val element = Order
    val tier    = session        // lives for the duration of the connection
    val size    = 4096
}
```

The compiler treats `OrderStream` as a `session`-tier channel with a coherency gap of `rtt_ns` ticks instead of L3 latency ticks. The write window must end at least `rtt_ns` ticks before the read window on the remote machine — the same proof obligation as any cross-core handoff, scaled to network latency. The compiler either satisfies it or rejects the program with a precise error: the declared RTT does not fit within the window budget.

The TCP connection state itself is a `session` channel — it lives for the duration of the connection and is reclaimed when the circuit that owns it is removed from the dispatch table. There is no separate connection object, no file descriptor, no socket buffer in kernel space. The connection is the channel subscription. When the remote endpoint closes the connection, the `NicCircuit` delivers a `channel TcpClose` signal to the owning circuit. The circuit handles it in its declared exhaustive match. The `session`-tier memory is reclaimed by the runtime at the owning circuit's removal — the same reclaim protocol as any other `session` value.

The consequence is that the distinction between "intra-machine microservices" and "inter-machine distributed systems" collapses to a single question: what is the declared backing and latency of the channel between them? Same machine, different cores: `session` tier, 50–200 ticks coherency gap. Different machines, same datacenter: `session` tier, network backing, 50,000 ns coherency gap. The circuit graph, the channel declarations, and the compiler's proof obligations are identical in both cases. Only the numbers change. A microservice is a circuit. Its interface is its declared `channel T` subscriptions. Its deployment is adding it to the live system. Its communication with other services is a memory handoff at a declared tick boundary.

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

```
channel RiskLimits {
    val element      = RiskLimitTable
    val tier         = session
    val size         = 1
    @Handoff(from = TradingEngine, to = RiskChecker, maxStaleness = "1ms")
}
```

Circuit diagram — two clocked blocks, one wire, one declared memory tier:

```
  core 1                                          core 2
  ┌─────────────────────────────┐                ┌─────────────────────────────┐
  │        TradingEngine        │                │         RiskChecker         │
  │  @Timeslice(core=1,period="10ns") │             │  @Timeslice(core=2,period="8ns") │
  │                             │                │                             │
  │  ┌──────┐                   │                │                   ┌───────┐ │
  │  │ ALU  │                   │                │                   │  ALU  │ │
  │  │trade ├──► write at       │                │    read at ◄──────┤ risk  │ │
  │  │logic │    window_end     │                │    window_start   │ check │ │
  │  └──────┘         │         │                │         ▲         └───────┘ │
  │                   │         │                │         │                   │
  │             ┌─────▼──────────────────────────────────┐│                   │
  │             │  channel RiskLimits {                   ││                   │
  │             │    val element = RiskLimitTable         ││                   │
  │             │    val tier    = session                ││                   │
  │             │    @Handoff(from = TradingEngine,       ││                   │
  │             │             to   = RiskChecker,         ││                   │
  │             │             maxStaleness = "1ms")       ││                   │
  │             │  }                                      ││                   │
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

### The Channel Model Forces Temporal Honesty

A consequence of the channel model that is easy to underestimate: **you cannot pretend that a network call is instantaneous.** In a conventional system, a programmer writes `response = await tcp_call(request)` — and the language allows it. The call may block for 50 ms. The thread sleeps, wakes, and continues. The temporal gap is hidden by the `await` keyword. The programmer's mental model is: I asked, I received. The actual model is: I asked, the thread was destroyed and recreated, and 50 ms later a different execution context received.

In the clock-aware model, there is no `await`. A circuit sends a request by writing to a `session`-tier TCP channel. It closes its window. The response, when it arrives, writes into a `channel TcpResponse`. A *separate circuit* — subscribed to `channel TcpResponse` — handles it at the next period boundary where data is available. The programmer cannot write a single circuit that sends and then waits. The model does not allow it. Waiting is not an operation in a window model; it is an absence of data in a channel.

This forces the programmer to declare the temporal structure of the operation explicitly:

```
circuit OrderSender {
    fn send(order: Order) -> () =
        OrderStream.put(order)     // write to TCP-backed channel; window closes
}

circuit OrderAck {
    fn onAck(ack: TcpResponse) -> AckResult =  // fires when response channel has data
        match ack {
            Accepted(id)   => log(id)
            Rejected(code) => handleRejection(code)
            Timeout        => handleTimeout()    // declared timeout channel fires separately
        }
}
```

The timeout is not a `try/catch` around a blocking call. It is a `channel Timeout` written by the `ClockCircuit` when the declared maximum response window expires. The `OrderAck` circuit subscribes to both `channel TcpResponse` and `channel Timeout` — whichever fires first is handled, and the match is exhaustive. The compiler refuses to compile an `OrderAck` that does not handle the timeout case. There is no way to write code that silently hangs waiting for a TCP response. The language structure makes hanging unexpressible.

### Two-Phase Locking Reduces to Channel State

The same temporal honesty makes distributed coordination trivially correct. A two-phase commit or two-phase lock in a conventional system requires a state machine, a timer, a timeout handler, a recovery path for coordinator failure, and careful consideration of all the intermediate states a participant can be in when a crash occurs. The complexity is not incidental — it comes from not knowing when things happen.

In the clock-aware model, a two-phase lock is a `session`-tier boolean channel:

```
channel TransactionLock {
    val element = LockState     // Unlocked | Locked(txId) | Committed | Aborted
    val tier    = session
    val size    = 1
}

circuit PhaseOne {
    fn acquire(req: LockRequest) -> LockState =
        TransactionLock.put(Locked(req.txId))   // set lock; window closes
}

circuit PhaseTwo {
    fn commit(decision: CommitDecision) -> LockState =   // fires when PhaseOne has written
        match decision {
            Commit => TransactionLock.put(Committed)
            Abort  => TransactionLock.put(Aborted)
        }
}

circuit ResourceGuard {
    fn onLockChange(state: LockState) -> () =   // subscribes to TransactionLock
        match state {
            Locked(txId)  => denyOtherRequests(txId)
            Committed     => applyAndRelease()
            Aborted       => rollbackAndRelease()
            Unlocked      => acceptRequests()
        }
}
```

`PhaseOne` writes `Locked` to `TransactionLock` and closes its window. `ResourceGuard` fires on the next period boundary, sees `Locked`, and begins denying competing requests — without any polling, without any mutex, without any kernel syscall. `PhaseTwo` fires when its input channel (the coordinator's decision) has data. It writes `Committed` or `Aborted` to `TransactionLock`. `ResourceGuard` fires again, sees the final state, and releases. Each transition is a channel write at a declared window boundary. Each handler is a circuit that fires exactly when data is available and never runs when it is not.

The timeout case — coordinator failure — is `channel Timeout` firing because `PhaseTwo`'s window was skipped too many times. The `ResourceGuard` match handles `Timeout` the same way it handles `Aborted`: rollback and release. There is no separate recovery thread, no lease renewal daemon, no heartbeat protocol. The clock is already in the model. The programmer declares the maximum wait in `system.cap` and the `ClockCircuit` writes the signal when it expires.

### Cryptographic Circuit Identity

Every compiled circuit manifest is signed with the organisation's private key before deployment. The signature covers the circuit's declared timing, lifetime types, channel subscriptions, and `@Handoff` declarations — the complete proof payload. The runtime verifies the signature before slotting the circuit into the dispatch table. An unsigned manifest, or one signed by an unknown key, is rejected at the dispatch boundary before any window is allocated, before any channel subscription is registered.

`@Handoff` extends this naturally. A handoff between two circuits is only permitted if both manifests are signed by keys within the same declared allocation:

```
channel RiskLimits {
    val element      = RiskLimitTable
    val tier         = session
    val size         = 1
    @Handoff(from = TradingEngine, to = RiskChecker,
             maxStaleness = "1ms", org = AcmeTradingCo)
}
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

