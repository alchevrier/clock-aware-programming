# ADR-0002: CPU Partition Model (OS Cores / App Cores)

**Date:** 2026-06-07  
**Status:** Accepted

---

## Context

The clock-aware programming model requires that a subset of CPU cores be free of OS-induced timing perturbations: scheduler preemption, interrupt delivery, RCU grace period work, and memory reclaim. Without this isolation, cycle counts on the hot path are not deterministic and cannot be verified at compile time.

Linux provides mechanisms for this isolation (`isolcpus`, `nohz_full`, `irqaffinity`, `rcu_nocbs`, NUMA policy), but they are independent knobs with no unified abstraction. A developer tuning an isolated core today configures four to six different subsystems separately, with no tool to verify that the configuration is complete or consistent.

The question is: what is the right architectural model for describing this split, and how should it be expressed?

---

## Decision

**Divide the system into two explicit partitions at the hardware topology level:**

- **OS partition** — cores 0–N reserved for the Linux kernel: scheduler, IRQ handling, RCU, memory reclaim, network stack (conventional path), and management plane processes.
- **App partition** — cores N+1–M reserved exclusively for the clock-aware workload: no scheduler intervention, no interrupt delivery, no RCU quiescent-state obligations, poll-mode I/O only.

**Boundary:** a shared ring buffer at a declared physical address, accessed by the OS partition at declared intervals and by the app partition at declared cycles. The ring buffer is the only legal data path between the two partitions.

---

## Rationale

### The FPGA precedent

Xilinx Zynq and Intel/Altera SoC devices already implement this split in silicon: the ARM Processing System (PS) runs Linux and handles management; the Programmable Logic (PL) fabric runs deterministic circuits. Data crosses the boundary via AXI streams — a declared-width, declared-rate channel. The PS does not directly modify PL registers mid-operation; the protocol boundary enforces this.

The CPU partition model is the software equivalent of this hardware split. The analogy is not decorative — it is the design rationale. A system already verified to work in hardware is being replicated in software under equivalent constraints.

### Determinism requires isolation, not optimisation

The standard approach to latency is optimisation: tune the scheduler, reduce lock contention, profile the hot path. Optimisation lowers average latency but does not bound worst-case latency because the sources of variance (interrupts, preemption, RCU, page faults) remain present at low frequency.

Bounding worst-case latency requires *removing* the variance sources from the hot path, not reducing their frequency. The partition model removes them structurally. An isolated core with `nohz_full` and no interrupt affinity cannot be preempted — not less often, but never.

### The ring buffer boundary is the clock domain crossing

In FPGA design, data crossing between clock domains requires explicit synchronisation (dual-port BRAM, handshake logic, FIFO with grey-coded pointers). The ring buffer boundary in the CPU partition model plays the same role. It is the *only* point where the two timing worlds meet. Both sides declare their access cycles; the compiler can verify that the OS partition's write cycle and the app partition's read cycle do not conflict.

### Cycle count becomes computable — and the ceiling is already published

Once a core is isolated, the microarchitecture model is fixed and the working set is pinned in L1 (< 32 KB). Instruction latency is deterministic under these conditions. `llvm-mca` already performs this analysis statically. The partition model creates the preconditions under which that static analysis is valid — without it, the analysis is aspirational.

Critically, the theoretical performance ceiling for any instruction sequence on any microarchitecture is not an empirical question — it is a published specification. Intel's Optimization Reference Manual, AMD's Software Optimization Guide, and ARM's Cortex technical reference manuals all document instructions per cycle per execution port, instruction latency, store-to-load forwarding latency, and cache access times per level. `llvm-mca` is a direct consumer of these documents. This means the compiler does not guess whether a declared budget is achievable; it checks the instruction mix against the published model and either verifies or rejects the annotation. The remaining variance between the model and the physical silicon — the gap the specification does not cover — is narrow, bounded, and well-characterised for server-class CPUs at pinned frequency on L1-resident data. It is handled by margin in the declared budget, the same way timing slack handles process variation in FPGA design.

The kernel itself is not a timing black box. The kernel is compiled code — known instruction sequences, known data structures, known call graphs. When the kernel is compiled with clock-aware annotations (or when a specific kernel code path is annotated as part of the `rust-for-linux` contribution in ADR-0003), that path's cycle count is just as computable as any user-space path. The kernel does not have special status in this model; it is source code targeting a known microarchitecture, and the same toolchain that verifies a user-space timeslice can verify a kernel code path. This is the mechanism by which the scheduler, RCU, and memory barriers are eliminated not just from the app partition but from the kernel itself: once the kernel's own hot paths are annotated, their timing is known, and every compensating mechanism that existed because that timing was unknown becomes redundant.

### The scheduler becomes unnecessary — everywhere

The Linux scheduler exists to arbitrate between tasks whose execution time is unknown at dispatch. Given that uncertainty, it must preempt: a task that runs longer than expected would otherwise starve everything else. The scheduler is a runtime solution to a compile-time unknowing.

This is not a property of user processes only. The kernel itself schedules internal work — softirqs, RCU callbacks, kthreads, workqueues — for the same reason: it does not know when that work will complete or conflict with other work. Every scheduling decision in the kernel, at every layer, is a runtime response to missing compile-time information.

Clock-aware annotations supply that information. When every kernel code path declares its core and its cycle window, the scheduler has no conflicts left to resolve. Not fewer conflicts — none. A fully annotated kernel does not need a preemptive scheduler any more than PL fabric needs one. The schedule is the program. `isolcpus` and `nohz_full` on the app partition are the first step; they demonstrate the property on a subset of cores. The claim is that the property holds universally when annotation coverage is complete.

### The unified result: a self-regulating circuit with architecture-aware timing

When the scheduler is unnecessary, RCU is unnecessary, locks are unnecessary, and the network stack is a declared ring buffer — what remains is a system that behaves as a circuit. Every operation's timing is known. Load changes are absorbed at the next cycle boundary, from within, without OS involvement. Configuration updates propagate through the config ring buffer at the declared cycle, not via a restart or a tuning session.

Critically, the timing is not hardcoded. `system.cap` declares `cpu_model = "skylake"`. The compiler derives cycle budgets from that model. Recompile with `cpu_model = "znver4"` and the budgets recompute — the same program, correct on different silicon, because the timing is a function of the architecture rather than a constant asserted against it. The same principle an XDC timing constraint file applies to FPGA synthesis: the target is declared, the toolchain proves it is met for the specific device and corner.

---

## Alternatives Rejected

### CPU affinity only (no full isolation)

Setting `taskset` or `pthread_setaffinity_np` pins a thread to a core but does not prevent the kernel from delivering interrupts or running RCU callbacks on that core. The result is lower average interference, not zero interference. Tail latency remains unbounded.

### DPDK-style "lcore" model

DPDK achieves core isolation at the application level via busy-polling on isolated cores, but it does not provide a compile-time model. The isolation is a runtime configuration; the compiler has no knowledge of it and cannot verify timing constraints against it. Clock-aware programming requires the partition to be a compile-time concept, not a deployment-time configuration.

### Separate physical machines

Running the OS workload on one machine and the trading workload on another eliminates OS interference but introduces network-path latency for any data that must flow between them. For a co-located strategy, kernel-bypass within a single machine is strictly faster. The partition model achieves the same isolation without a physical boundary.

### PREEMPT\_RT kernel

A real-time kernel reduces preemption latency and improves interrupt response but does not eliminate scheduler intervention on the hot core. PREEMPT\_RT is an improvement to the OS partition; it does not create an app partition. Clock-aware programming targets a harder guarantee than RT kernels provide.

---

## Consequences

- The partition model is the precondition for all other decisions in this ADR set. Without it, compile-time cycle verification is invalid.
- Every annotated function must declare which partition it executes in. The compiler rejects `[[timeslice(...)]]` annotations on code that can execute on OS-partition cores.
- The ring buffer boundary becomes the sole interface specification between the two execution worlds. It must declare: physical address, size, producer core, consumer core, producer write cycle, consumer read cycle.
- System configuration becomes a single file (see ADR-0005) that generates the kernel boot parameters for both partitions.
