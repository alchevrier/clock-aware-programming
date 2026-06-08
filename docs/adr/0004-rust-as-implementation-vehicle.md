# ADR-0004: Rust as Implementation Vehicle

**Date:** 2026-06-07  
**Status:** Accepted

---

## Context

Clock-aware programming requires a language that can:

1. Express timing constraints as verifiable annotations, not comments.
2. Enforce the constraint at compile time, not at runtime.
3. Integrate with the Linux kernel (for the RCU elimination contribution in ADR-0003).
4. Support low-level, zero-overhead, `unsafe` operations where hardware interaction is required.

The choice of language determines what the annotations look like, what the compiler can prove, and whether the contribution has a viable path to kernel mainline.

---

## Decision

**Rust is the implementation vehicle for clock-aware annotations, the compile-time checker, and the kernel contribution.**

Specifically:

- Clock-aware annotations are expressed as Rust attributes: `#[timeslice(core = 2, cycle = N, budget_ns = 4)]`.
- The compile-time checker is a proc-macro crate that verifies annotated accesses do not produce overlapping cycle windows.
- The `#[no_rcu]` attribute is valid only when the checker passes.
- The kernel contribution targets `rust-for-linux`, where Rust is already an accepted language with Linus Torvalds' explicit support.

---

## Rationale

### The verification model: four rules against the declared budget

When the proc-macro encounters a `#[timeslice(budget_ns = B)]` annotation on a function, it compiles the function body, invokes `llvm-mca` with the declared `cpu_model`, and evaluates four rules against the declared budget. All four must pass for the annotation to be accepted. Any failure is a compile error.

**Rule 1 — ALU cost.** For each instruction in the compiled function body, the checker looks up its latency and throughput from the microarchitecture model (Intel Optimization Reference Manual, AMD Software Optimization Guide, or ARM Cortex TRM — all consumed by `llvm-mca`). The sum of ALU instruction latencies along the critical path must not exceed the declared budget:

```
Σ(latency(instr_i) for instr_i on critical path) ≤ budget_cycles
```

**Rule 2 — Memory access cost.** Load and store instructions have declared latency from the cache hierarchy. For an L1-pinned working set, load latency is 4–5 cycles (L1 hit), store latency is 1 cycle (store buffer). The checker verifies that all memory accesses in the function are to declared working-set addresses and their latency is accounted for in the budget. An access that would miss L1 — because the address is outside the declared working set — is a compile error, not a runtime cache miss.

**Rule 3 — Port contention.** Modern CPUs have a fixed number of execution ports. Each instruction type is dispatched to a specific port (or set of ports). If two instructions in the same cycle window compete for the same port, one must wait — adding cycles to the actual execution time that the nominal latency sum does not capture. `llvm-mca` models port pressure precisely per microarchitecture. The checker verifies that the instruction mix in the function does not create port contention that causes the actual throughput to exceed the declared budget. If it does, the annotation is rejected: the programmer must either widen the budget, reorder the instruction mix, or add a `#[pipeline]` annotation to spread the work across stages.

**Rule 4 — Pipeline initiation interval.** For functions annotated with `#[pipeline(ii = N)]`, the checker verifies that the declared II is achievable: that no loop-carried dependency chain is longer than N cycles, and that port utilisation across the II window does not exceed 100% on any port. This is the direct equivalent of HLS II verification. A function that declares `ii = 1` but has a 3-cycle loop-carried dependency is rejected at compile time — not discovered at simulation, not measured at runtime.

Together these four rules implement the same verification that Vivado performs during HLS synthesis: given this microarchitecture model, given this instruction mix, given these resource constraints — does the declared timing close? The answer is yes or no, at compile time, with a precise error message identifying which rule failed and by how many cycles. The programmer fixes the code or widens the budget. The process is timing closure, in software.

### Code structure mirrors SystemVerilog dataflow — channels as wires, functions as modules

SystemVerilog describes hardware as dataflow: modules consume input ports and produce output ports, connected by wires. The structure is declarative — the designer describes what flows where, not how the scheduler moves data. A NASDAQ feed handler in SystemVerilog or HLS is a pipeline of stages connected by channels:

```
parse_event → handle_event → update_snapshot
```

Clock-aware Rust expresses the same structure identically:

```rust
#[timeslice(core = 2, cycle = N, budget_ns = 4)]
#[pipeline(stages = 3, ii = 1)]
fn kernel(raw_in: &Channel<RawMessage>, out: &Channel<BookSnapshot>) {
    update_snapshot(handle_event(parse_event(raw_in)), out);
}
```

`Channel<T>` is a typed, declared-timing conduit — not a thread-safe queue with a lock, not an async future with a waker, but a direct analogue of an HLS `hls::stream<T>`: a channel whose read and write cycles are declared and verified by the timeslice checker. `parse_event`, `handle_event`, and `update_snapshot` are pipeline stages, each with its own `#[timeslice]` annotation declaring its cycle budget. Their composition — `update_snapshot(handle_event(parse_event(...)))` — is the data path, expressed as function composition exactly as it would be expressed in HLS.

The consequence: a clock-aware Rust program reads like a hardware description, because it *is* a hardware description — of the architecture above the silicon. The compiler is the synthesis tool. The annotations are the constraints. The channel types enforce that data flows only between declared-compatible cycle windows, the same way HLS FIFO depth constraints enforce that a producer does not overflow a consumer. The mental model the programmer uses is identical to the mental model used in Vitis HLS — because the underlying abstraction is the same: a pipeline of typed stages connected by declared-timing channels, verified at compile time.

This is not a stylistic preference. It is the correct abstraction for a clock-aware system. Software written in this style is directly portable to an in-order clock-aware CPU (ADR-0009) because it already describes a pipeline with declared throughput — the hardware just needs to execute what the compiler has already proven correct.

### Proc-macro annotations mirror HLS directives — and that is intentional

HLS tools (Vitis HLS, Bambu, Catapult) use directives — pragmas or attributes placed in source code — to declare pipeline behaviour, initiation interval, loop unrolling, and resource allocation. These are not hints; they are constraints the synthesis tool verifies and enforces. A function marked `#pragma HLS PIPELINE II=1` that cannot achieve II=1 fails synthesis. The pragma is a contract, not a comment.

Rust proc-macros implement the same model for software:

```rust
#[timeslice(core = 2, cycle = N, budget_ns = 4)]
#[pipeline(stages = 3, ii = 1)]
fn process_order(book: &OrderBook) -> Decision { ... }
```

`#[timeslice]` declares when this function executes and for how long — equivalent to HLS `set_max_delay` and `PIPELINE II`. `#[pipeline]` declares the number of pipeline stages and the initiation interval — equivalent to HLS loop pipelining directives. Both are compile-time contracts: the proc-macro invokes `llvm-mca` on the compiled function body, verifies the declared cycle budget and II against the actual instruction mix, and emits a compile error on violation. Not a warning. Not a runtime assertion. A compile error — the same enforcement model as HLS timing closure.

The parallel is not superficial. HLS directives work because the synthesis tool has full visibility into the hardware schedule. Rust proc-macros work because the compiler has full visibility into the instruction schedule. In both cases, the annotation is the contract, the tool is the verifier, and violation is a build failure. The discipline that Vivado enforces on FPGA designs is exactly what this annotation system enforces on software — the same contract, the same enforcement, different silicon.

`#[pipeline]` in particular makes the FSM execution model of ADR-0002 concrete. A pipeline with declared stages and II=1 is a software pipeline that processes one new input per cycle — the same initiation interval guarantee that HLS achieves in hardware. The compiler proves the instruction mix fits within the declared stage count and produces no structural hazards. The result is a software function with the throughput properties of an FPGA pipeline, running on a CPU that trusts the declared schedule.

### The borrow checker already proves the invariant RCU enforces

Rust's borrow checker proves read/write exclusivity at compile time: a mutable reference (`&mut T`) cannot coexist with any other reference to the same data. This is structurally the same guarantee RCU provides at runtime — a reader will not observe a partially-written value.

Adding timeslice annotations extends this existing proof system. The borrow checker proves *spatial* exclusivity (no aliased mutable references); the timeslice checker proves *temporal* exclusivity (no overlapping cycle windows). The combination eliminates the need for the runtime RCU mechanism entirely.

Building on an existing, sound proof system is feasible. Building a new one from scratch in C is not — C has no ownership model for the type system to extend.

### Lifetimes are a natural hook for timeslice semantics

Rust lifetimes encode validity ranges for references. A timeslice annotation fits naturally as a lifetime bound:

```rust
#[timeslice(core = 2, cycle = N, budget_ns = 4)]
fn read_order_book(book: &OrderBook) -> BestBid { ... }
```

The annotation reads: "this reference is valid during cycle N on core 2." The type system can reason about whether two such references with different cycle annotations can coexist — which is exactly the proof needed for RCU elimination.

### `unsafe` boundary already exists for hardware-level operations

Clock-aware code that directly accesses hardware-mapped memory (NIC ring buffers, MMIO registers) belongs in `unsafe` blocks. Rust's `unsafe` boundary is precisely scoped: the compiler verifies everything outside `unsafe`; the programmer takes responsibility for invariants inside it. This is the right model for hardware interface code — it does not pretend safety where it cannot be proven, and it does not sacrifice the proof for the rest of the codebase.

Clock-aware annotations that govern hardware access sit at this boundary. The annotation is the declaration of the invariant; the `unsafe` block is where the hardware access happens; the checker verifies the annotation is satisfied before permitting the `unsafe` code.

### `rust-for-linux` is a real community path

C is the established language of the kernel. A C extension or C preprocessor macro scheme for clock-aware annotations would need to be proposed as a new kernel subsystem — a high bar with no existing constituency.

Rust-for-Linux is different:

- Linus Torvalds has explicitly accepted Rust as a second implementation language for kernel drivers and subsystems.
- The `rust-for-linux` maintainers are actively soliciting cases where Rust's type system eliminates kernel synchronisation primitives.
- A proc-macro crate implementing the timeslice checker can be developed, tested, and reviewed entirely outside the kernel tree, then presented to the kernel community with a working proof-of-concept patch.

The contribution path is: proc-macro crate (standalone) → kernel patch replacing one RCU critical section → RFC to `rust-for-linux` mailing list. This is a real sequence with known actors, not a theoretical path.

### Proc-macros ship as an independent crate — no Rust committee involvement

This is a practical and significant advantage of the proc-macro approach. A proc-macro is a compiler plugin expressed as a standard Rust crate. It lives in its own repository, has its own version, compiles against stable Rust, and is distributed via `crates.io`. There is no language change proposal, no RFC to the Rust lang team, no waiting for stabilisation, no committee review.

The timeslice checker, the `#[timeslice]`, `#[pipeline]`, and `#[no_rcu]` attributes, the `llvm-mca` integration, and the `Channel<T>` type are all implementable as a crate today, on stable Rust, independently of any language evolution. Users add the crate as a dependency and the annotations become available immediately. If the design evolves — new attributes, refined verification rules, updated microarchitecture models — the crate is updated and versioned independently.

This is the right delivery model for a new primitive: prove it works as a library before proposing it as a language feature. If the crate gains adoption and the community finds the model sound, the path to a formal language proposal opens from a position of demonstrated utility rather than theoretical argument. The proc-macro is not a workaround — it is the correct first step.

### Crates declare their timing contracts — incompatible crates are rejected at compile time

A clock-aware crate is not just a library — it is a timing contract. A crate that exposes functions annotated with `#[timeslice]` declares, as part of its public API, the cycle budget and core assignment those functions require. Any crate that depends on it must declare compatible timing — if the dependency's declared cycle window overlaps with the caller's window, or if the dependency's budget exceeds the caller's remaining budget, the timeslice checker rejects the dependency at compile time.

This extends the proc-macro verification across the full dependency graph. In `Cargo.toml`:

```toml
[dependencies]
order-book = { version = "1.2", timeslice = { core = 2, max_budget_ns = 2 } }
```

The `timeslice` field is a compile-time constraint on the dependency: "I will use this crate from core 2 and it must fit within 2 ns." If `order-book 1.2` declares its hot path at 2.5 ns, the dependency is rejected — not at link time, not at runtime, but when Cargo resolves the dependency graph. The error is: "order-book 1.2 declares budget_ns = 2.5, which exceeds the declared constraint of 2.0 for core 2."

The practical consequence is that the crate ecosystem becomes timing-safe by construction. A crate that ships a timing regression — a new version whose hot path exceeds its declared budget — cannot be pulled into a clock-aware binary without an explicit budget acknowledgement from the consumer. This is the same guarantee Rust's type system provides for memory safety, extended to timing: an incompatible dependency is a compile error, not a runtime surprise discovered in production at 3am.

The dependency resolver also propagates the budget constraint transitively: if crate A depends on crate B which depends on crate C, the checker verifies that the sum of declared budgets across the call chain fits within the top-level timeslice window. A chain that exceeds the budget is rejected at the root, with a precise error identifying which dependency in the chain caused the violation and by how many cycles.

### The build produces a compliance artefact automatically

For safety-critical systems — DO-178C (avionics), IEC 61508 (industrial safety), AUTOSAR (automotive), or any domain requiring formal worst-case execution time evidence — the certification body requires proof that the declared timing is met. Today that proof is produced by external WCET analysis tools (AbsInt aiT, RapiTime, Bound-T) running against a pre-compiled binary on a specific hardware configuration. The proof is post-hoc, binary-specific, and must be regenerated for every code change and every target hardware revision.

Under the clock-aware model, the proof is a build output. The proc-macro checker, as a by-product of compilation, emits a structured timing report: for each annotated function, the declared budget, the verified worst-case cycle count from `llvm-mca`, the margin, the core assignment, the memory access profile, and the dependency chain. This report is not a test result — it is a compile-time certificate. It cannot be produced if the build fails, and the build fails if any timing constraint is violated. The certificate and the binary are produced by the same invocation and are structurally inseparable.

No modification to Cargo or rustc is required. The mechanism uses two extension points that already exist:

1. **Proc-macro side effects.** A proc-macro executes at compile time during macro expansion and has full access to `std::fs`. For every annotated function, the timeslice checker writes a per-function timing record — declared budget, verified worst-case cycle count, margin, core assignment, memory profile — to `target/timeslice/<crate>/<function>.json`. This is a side effect of the normal `cargo build`.

2. **Cargo subcommand extensibility.** Any binary named `cargo-X` on `$PATH` is invocable as `cargo X` without any Cargo modification. The second crate in the ecosystem, `cargo-timeslice`, is a CLI binary that reads `target/timeslice/` and aggregates the per-function records into the final report. Invoked as:

```
cargo timeslice report
```

The output is a machine-readable (JSON/TOML) and human-readable (Markdown) document stating: "function `process_order` on core 2, declared budget 4 ns, verified worst-case 3.1 ns, margin 0.9 ns, llvm-mca model skylake, dependency chain [parse_event: 1.2 ns, handle_event: 0.9 ns, update_snapshot: 1.0 ns], L1-pinned working set confirmed." This document is the certification artefact. It is reproducible — the same source, the same `cpu_model`, and the same crate versions always produce the same report. It is traceable to source — every annotated function in the report corresponds to a specific line in a specific file at a specific commit. It is complete — every timing-annotated code path in the binary is covered, with no unannotated paths permitted in `#[no_rcu]` or `#[no_barrier]` contexts.

For DO-178C Level A software, this replaces a significant fraction of the structural coverage analysis and WCET evidence that currently requires specialised tooling and manual review. The compiler is the analysis tool, the source annotation is the specification, and the build is the proof. The certification artefact is not generated after the fact — it is a direct output of the development process.

### The timing contract is cryptographically sealed — it cannot be spoofed at load time

The compliance artefact alone does not prevent a tampered binary from claiming a timing contract it did not compile against. A binary that was recompiled with wider cycle budgets, or a manifest that was edited after the fact to show smaller worst-case numbers, would silently pass a naive load-time check. The cryptographic seal closes this gap.

At build time, the proc-macro assembles a **signed manifest** embedded in the binary. The manifest contains:

- The application name and declared subscription list
- The full timing contract: every annotated function, its declared budget, its verified worst-case cycle count, its core assignment, and its margin
- The SHA-256 hash of the compiled binary

The build system signs this manifest with an Ed25519 private key held in CI. The corresponding public key is pinned in `system.cap`:

```toml
[system]
trusted_build_key = "ed25519:AAAA..."
```

The loader verifies the manifest signature against this pinned public key before mapping any segment of the binary. An attacker who wants to run a binary with a falsified timing contract faces an impossible constraint: they must produce a manifest that (a) contains the falsified cycle budgets, (b) contains the correct SHA-256 hash of their binary, and (c) is signed by the CI private key — which they do not have. A binary recompiled with looser budgets will have a valid content hash but no valid signature. A manifest copied from a legitimate binary will have a valid signature but the wrong content hash. There is no combination that passes all three checks without the private key.

The private key never leaves CI. Key rotation is a one-line change to `system.cap` — a visible, reviewable diff in version control. There is no runtime trust negotiation, no certificate authority, no OCSP check. The trust chain is: source → CI build → signed manifest embedded in binary → pinned public key in `system.cap` → loader verification → execution. Every link is static and produces a visible artefact. The timing contract that the compliance document reports is the same timing contract the loader enforces — cryptographically identical, by construction.

**This is the proof the certification body receives.** A DO-178C auditor, an IEC 61508 assessor, or a regulator reviewing the system does not have to trust that the compliance document was generated honestly or that it accurately reflects the deployed binary. They verify the Ed25519 signature on the manifest against the published public key. If the signature is valid, three things are simultaneously proven:

1. The binary was built by the authorised CI system holding the private key — it was not compiled by an attacker, a developer workstation, or an unknown toolchain.
2. The timing contract in the manifest — every declared budget, every verified worst-case cycle count, every margin — was produced by the same build that produced the binary. It was not edited after the fact.
3. The binary has not been modified since it was signed — any tampering invalidates the content hash inside the manifest, which invalidates the signature.

The compliance document is not a report that claims the binary meets its timing contract. It is a cryptographically verifiable statement that the binary was built correctly and the stated numbers are unfalsified. The auditor's job, which today involves re-running WCET tools, comparing tool outputs to source, and tracing the chain from specification to binary, collapses to a single signature verification. The proof is machine-checkable, reproducible, and requires no trust in the developer or the development process — only in the public key, which is version-controlled and auditable at every rotation.

**The kernel itself becomes compliant.** The clock-aware model does not stop at the application boundary. If the kernel's own circuits — IRQ handlers, RCU reclaim, memory management, the management plane — carry `#[timeslice]` annotations and are built by the same proc-macro checker, the kernel binary receives the same signed manifest with the same verified timing contract. The kernel is a clock-aware application in its own right. The certification body receives a cryptographically verifiable timing proof not just for the application, but for the entire software stack from kernel circuit to application circuit. There is no trusted black box beneath the verified code. Every annotated function in every layer — kernel, runtime, application — is covered by the same proof, bound into the same signed manifest, verifiable with the same public key. For safety-critical domains (avionics, automotive, industrial), this closes the gap that has always existed between "we verified the application" and "we trust the OS beneath it." Under clock-aware programming, trust is not assumed for the OS — it is proven, with the same mechanism, at the same build step.

**This is something that has never existed in software engineering.** Every programming discipline that calls itself "hardware-aware" — DPDK, XDP, HFT tuning, real-time systems, kernel bypass networking — reasons about the hardware directly: cache line sizes, NUMA domains, execution port pressure, memory access latency. But every one of these disciplines treats the OS as a ghost in the model: present, necessary, but outside the proof. The programmer knows the CPU executes in 4 cycles and the NIC DMA completes in 200 ns, but the OS is an assumption — "it won't preempt here because we set `isolcpus`", "the IRQ won't fire here because we set `irqaffinity`", "the RCU grace period won't stall here because we set `rcu_nocbs`". These are configuration disciplines that approximate what a proof would provide. They are not proofs.

Clock-aware programming closes the stack. The hardware timing model (the microarchitecture spec consumed by `llvm-mca`) is the foundation. The OS circuits above it are verified against that model with `#[timeslice]` annotations. The application circuits above those are verified against the same model. The signed manifest covers all three layers. The certification body sees a single, coherent, cryptographically verifiable proof that spans from the silicon's timing behaviour to the application's declared cycle budgets — with the OS not assumed away but verified in place.

The OS is not absent from the model, as it has always been in hardware-aware programming. It is not a trusted black box, as it has always been in safety-critical certification. It is a first-class participant in the proof — annotated, verified, and sealed into the same manifest as the hardware and the application. This is the property that makes the entire stack provably correct for the first time.

---

## Alternatives Rejected

### C with preprocessor macros

C has no compile-time type system extension mechanism beyond macros and `_Generic`. A clock-aware annotation in C would be a macro that expands to a runtime assertion at best, or a comment at worst. Neither provides a compile-time proof. The kernel is C; a C implementation has zero friction for upstreaming, but it cannot deliver the core guarantee (compile-time verification) that makes the model useful.

### C++ attributes (`[[timeslice(...)]]`)

C++ attributes can trigger compiler plugin behaviour. `llvm-mca` already consumes C++ source to perform static throughput analysis. A Clang plugin that processes `[[timeslice(...)]]` is technically feasible. The problem: the Linux kernel does not use C++ and is not open to C++ contributions. The plugin approach has no path to kernel integration and would require a separate ABI boundary. Useful for userspace; irrelevant for the kernel contribution.

### A new language

A purpose-built language with clock-aware semantics as a first-class primitive would be the ideal end state. It is not the right starting point: no compiler, no toolchain, no community, no kernel integration path. It would take the concept from "publishable whitepaper" to "years of compiler infrastructure work" with no guarantee of adoption. The decision is not about what the ideal tool would be; it is about what can deliver a credible, verifiable, kernel-adjacent implementation in the near term.

### Zig

Zig has excellent comptime evaluation and zero-overhead `unsafe`-equivalent semantics. It does not have a Rust-equivalent ownership model, so the proof of read/write exclusivity that Rust's borrow checker provides does not exist. More importantly, Zig has no kernel integration path comparable to `rust-for-linux`. The comptime evaluation is powerful but would require building the cycle proof system from scratch rather than extending an existing sound type system.

---

## Consequences

- The proc-macro crate is the first concrete deliverable. It must expose `#[timeslice(...)]` and `#[no_rcu]` attributes and emit compile errors on violation.
- The crate requires access to the system configuration (ADR-0005) at compile time to know core assignments and cycle budgets.
- The kernel contribution is a Rust patch, not a C patch, and targets `rust-for-linux` drivers or subsystem code — not the core RCU implementation.
- If `rust-for-linux` grows to cover scheduler or networking subsystems before this contribution is ready, the target code path for the proof-of-concept patch should be updated accordingly.
