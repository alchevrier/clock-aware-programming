# ADR-0005: Unified System Configuration File

**Date:** 2026-06-07  
**Status:** Accepted

---

## Context

Isolating CPU cores for the app partition (ADR-0002) currently requires configuring multiple independent Linux subsystems:

| Subsystem | Mechanism | Location |
|---|---|---|
| Scheduler isolation | `isolcpus=2-7` | Kernel boot parameters (`/etc/default/grub`) |
| Timer tick suppression | `nohz_full=2-7` | Kernel boot parameters |
| IRQ affinity | `irqaffinity=0,1` | Kernel boot parameters |
| RCU callback offload | `rcu_nocbs=2-7` | Kernel boot parameters |
| RCU nocb boost | `rcu_nocb_poll` | Kernel boot parameters |
| NUMA memory policy | `numactl --membind` | Process launch scripts |
| Huge page allocation | `vm.nr_hugepages` | `sysctl` / `sysctl.conf` |
| CPU frequency pinning | `cpupower frequency-set` | Init scripts |
| IRQ SMP affinity | `/proc/irq/*/smp_affinity` | Post-boot scripts |

These mechanisms are designed by different kernel subsystems, configured in different files, at different times in the boot sequence, with no shared schema and no cross-validation. A misconfiguration in one (e.g., forgetting `rcu_nocbs`) leaves the rest correct but produces the wrong aggregate behaviour — and the error is silent at configuration time, visible only as latency spikes at runtime.

Additionally, the compiler's timeslice verification (ADR-0003) needs to know the partition layout at compile time: which cores are app-partition cores, what microarchitecture they run, and what the declared ring buffer access cycles are. Today, this information does not exist in any machine-readable form. The developer holds it in their head and translates it manually into boot parameters on one side and magic cycle constants on the other.

---

## Decision

**A single declarative configuration file (`system.cap`) describes the full partition layout. Code generation derives kernel boot parameters and scopes the compiler's verification domain from this file.**

Example schema (TOML):

```toml
[system]
cpu_model = "skylake"

[os_partition]
cores = [0, 1]

[app_partition]
cores = [2, 3, 4, 5, 6, 7]
working_set = "L1"        # compiler enforces: declared data must fit in L1
huge_pages = 512          # 1 GB total (512 × 2 MB)
cpu_freq_mhz = 3600       # pinned; no turbo, no power states

[boundary]
ring_buffer_paddr = "0x200000000"   # physical address, 2 MB aligned
ring_buffer_size  = "2MB"
os_write_cycle    = 1000            # OS partition writes every 1000 cycles
app_read_cycle    = 500             # app partition reads every 500 cycles
```

From this file:

- A code generator emits the full kernel boot parameter string.
- The Rust proc-macro crate reads the file at compile time to validate `#[timeslice(...)]` annotations against declared core assignments and cycle budgets.
- The ring buffer boundary contract is machine-readable and checked against both sides.

---

## Runtime Extension: Self-Regulating Configuration Library

`system.cap` defines the initial partition layout and scopes the compiler's verification domain. At runtime, a system library (`libcap` or equivalent) extends this with dynamic reconfiguration.

### Mechanism

The system is already self-regulating at the poll cycle boundary (ADR-0006): the app partition reads the ring buffer head at a declared cycle interval, processes all available data, then loops. This loop boundary is the natural point at which the system can also check for configuration updates:

```
loop:
  head = ring.head.load(Relaxed)
  if head != ring.tail:
      process_batch(ring, head)
      ring.tail.store(head, Release)

  // self-regulation: poll rate adapts to ring depth
  // configuration sync: new params take effect here, at the cycle boundary
  if config_ring.has_update():
      apply_config(config_ring.read())

  wait_cycles(next_poll)
```

The OS partition writes new configuration parameters into a dedicated config ring buffer (a second boundary channel, declared in `system.cap`). The app partition reads and applies them at the next cycle boundary. Because the cycle boundary is a declared, compiler-known point in time, the configuration update is not a surprise — it is a scheduled transition.

### What can be updated at runtime

- Cycle budgets per annotated function (tightening or relaxing latency targets).
- Poll backoff thresholds (adapting to load patterns observed by the OS partition).
- Working set hints (prefetch patterns, cache residency targets).
- Core assignment changes — adding or removing cores from the app partition — take effect at the next cycle boundary after the kernel-side configuration is updated via the library.

### What cannot be updated at runtime

Compile-time annotations (`#[timeslice(...)]`) are baked into the binary. A budget declared at compile time cannot be widened at runtime beyond what the compiler verified. The runtime library can only operate within the envelope the compiler has already proved safe. It can narrow budgets (more conservative), not widen them (which would invalidate the compile-time proof).

### Library interface (sketch)

```rust
// OS partition side — called from management plane
cap::update_poll_backoff(core: u32, high_watermark: u64, low_watermark: u64);
cap::update_cycle_budget(fn_id: FnId, budget_cycles: u64);
cap::add_core_to_app_partition(core: u32);  // kernel-side isolation applied first

// App partition side — called automatically at cycle boundary
// Not exposed directly; integrated into the generated poll loop
```

---

## Rationale

### A single source of truth eliminates silent misconfiguration

Today, the developer translates a mental model (core 2 is isolated, ring buffer is at this address) into four to six separate configuration locations. Each translation is a potential error. Errors are not caught until runtime — and often not immediately at runtime, appearing as rare latency spikes under load.

A single file with a schema makes the configuration complete-or-not-compile. If `app_partition.cores` is not declared, the compiler cannot validate any `#[timeslice(core = 2, ...)]` annotation and refuses to compile. The error surface moves from runtime latency histograms to compile-time diagnostics.

### The compiler needs this information anyway

Clock-aware annotations reference core numbers and cycle budgets. The compiler must know: does core 2 belong to the app partition? What is the declared CPU frequency on that core? What is the ring buffer access cycle? This information cannot be embedded in the annotation alone — it would create duplication and inconsistency between the annotation and the actual system configuration.

The system configuration file is where this information lives canonically. The compiler reads it; the annotations reference it; the boot parameter generator consumes it. One schema, three consumers, zero duplication.

### Code generation for boot parameters is already established practice

Device tree overlays, Vivado XDC constraint files, and systemd unit generators all use code generation from a higher-level description to produce lower-level configuration. Kernel boot parameters are not special — they are a text file consumed at boot. Generating them from a schema is straightforward and eliminates manual translation errors.

### Scope for `llvm-mca` integration

The declared `cpu_model` in the config file determines which `llvm-mca` target is used for cycle count analysis. Without this declaration, the developer must specify the target separately for `llvm-mca` invocations and hope it matches the actual hardware. With the config file, the build system uses the declared model consistently.

### Memory ordering is eliminated by default

Memory barriers — `acquire`, `release`, `seq_cst`, `smp_mb()`, `READ_ONCE`, `WRITE_ONCE` — exist because the CPU and compiler reorder instructions, and without declared access timing, the programmer must constrain that reordering at runtime to prevent data races. Every barrier is a runtime assertion that a particular ordering must hold, inserted because the compiler does not know whether it does.

When `system.cap` declares every access cycle for every core, the compiler knows the ordering of every operation across every core at compile time. If core 2 reads at cycle N and core 0 writes at cycle N−K where K > 0, the write provably precedes the read — not because a barrier enforced it, but because the schedule makes it impossible for it not to. The barrier is not needed. Not elided as an optimisation — unnecessary by construction, the same way a lock between two operations that can never overlap is unnecessary by construction.

This eliminates memory ordering overhead at every layer: kernel hot paths, userspace ring buffers, lock-free queues, RCU read-side critical sections. Every `smp_mb()` in the kernel, every `std::atomic` with `memory_order_acquire` in userspace, every `rcu_dereference()` — each is a runtime substitute for a compile-time proof the system does not have. `system.cap` is where that proof is grounded: the timing declarations it contains are the axioms from which the compiler derives ordering for every annotated access.

---

## Alternatives Rejected

### Continue with hand-written boot parameters

Maintainable for one developer on one machine. Not reproducible across machines, not validated at build time, and not readable by the compiler. The core problem — that the partition layout is not a machine-readable artifact — is not addressed.

### Environment variables at build time

`CAP_APP_CORES=2-7 cargo build` avoids a config file. Environment variables have no schema, no validation, no documentation, and no natural place to express compound types (ring buffer address + size + access cycle as a unit). They are a viable hack for a single constraint; they do not compose for a full partition declaration.

### Kernel cgroup / cpuset interface

Linux cgroups can express CPU and memory affinity. They are a runtime interface, not a compile-time schema. The compiler cannot read a cgroup configuration. Additionally, cgroups do not cover `nohz_full`, `irqaffinity`, or `rcu_nocbs` — they address scheduling affinity only, not the full isolation stack.

### Systemd unit files with CPU affinity

Systemd `CPUAffinity=` and `CPUSchedulingPolicy=` configure a process's CPU affinity. They do not generate kernel boot parameters, do not suppress timer ticks (`nohz_full`), do not re-route IRQs, and are not readable by the compiler. They address one layer of the isolation stack, not all of them.

---

## Consequences

- `system.cap` (or equivalent filename) is a required artifact in any clock-aware project. Its absence is a compile error, not a warning.
- The proc-macro crate that implements `#[timeslice(...)]` reads `system.cap` at compile time via `include_str!` or a build script. The path to `system.cap` is a build-time constant.
- A code generator (`cap-gen` or similar) takes `system.cap` and emits: kernel command line additions, a `sysctl.conf` fragment, and a `cpupower` invocation script.
- The ring buffer physical address in `system.cap` must match the `mmap` call in application code — the build system validates this if the application declares its ring buffer address as a constant derived from the config.
- On machines where the declared CPU model does not match the actual CPU (`/proc/cpuinfo`), the build system emits a warning. It is the developer's responsibility to ensure the declared model is correct; the compiler cannot detect this at compile time.
- A second ring buffer channel (the config ring) is declared in `system.cap` alongside the data ring. Its address and size are also compile-time constants. The runtime library writes to it from the OS partition; the generated poll loop reads from it at the cycle boundary.
- Runtime configuration updates are bounded by the compile-time proof envelope: they can narrow cycle budgets but not exceed them.
