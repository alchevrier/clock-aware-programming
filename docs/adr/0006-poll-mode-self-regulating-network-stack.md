# ADR-0006: Poll-Mode Self-Regulating Network Stack

**Date:** 2026-06-07  
**Status:** Accepted

---

## Context

The Linux network stack is structured around interrupt delivery:

```
NIC receives packet
  → hardware raises hardirq
  → hardirq handler schedules NAPI softirq
  → softirq runs on some CPU
  → sk_buff allocated from slab
  → packet copies through socket layers
  → application read() / recvmsg()
```

Every step in this chain introduces non-determinism:

- **hardirq** — can be delivered to any core at any time, including app-partition cores, causing preemption.
- **softirq** — scheduled by the kernel; timing depends on the scheduler's decision about when to run the softirq vector.
- **sk_buff allocation** — `kmalloc` on the hot path; allocation latency spikes under pressure.
- **Socket layer copies** — data copied from NIC buffer → sk_buff → socket receive buffer → application buffer. At minimum two copies; sometimes three.
- **Syscall boundary** — `recvmsg()` is a context switch. On a modern CPU with KPTI, a syscall costs 100–500 ns even on a cold cache.

For a latency-sensitive application, each of these is a source of variance, not a source of cost. The average cost is low; the tail cost is unbounded.

DPDK and XDP address this by bypassing the kernel stack entirely. DPDK runs the NIC driver in userspace as a busy-polling loop. XDP runs an eBPF program in the kernel before `sk_buff` allocation. Both eliminate the interrupt-driven path.

However, DPDK and XDP are runtime configurations. The compiler has no knowledge of the poll loop, no access to the declared poll cycle, and cannot integrate network stack behaviour into timeslice analysis.

---

## Decision

**The network stack in the app partition is a memory-mapped ring buffer polled at a declared cycle interval. The poll rate self-regulates on ring depth. The entire mechanism is declared in `system.cap` and is visible to the compiler.**

### Mechanism

The NIC DMA engine writes received packets directly into a pre-allocated ring buffer at a declared physical address (the same address declared in `system.cap` under `[boundary]`). The app-partition hot loop polls the ring buffer's head pointer at a declared cycle interval:

```
loop:
  head = ring.head.load(Relaxed)      // one cache-line read, no lock
  if head != ring.tail:
      process_batch(ring, head)        // process all available packets
      ring.tail.store(head, Release)
  // self-regulation:
  next_poll = ring.depth() > THRESHOLD ? NOW : NOW + BACKOFF_CYCLES
  wait_cycles(next_poll)
```

The poll cycle (`500` in the `system.cap` example in ADR-0005) is a declared constant. The compiler knows the app partition reads the ring buffer every 500 cycles. This is the same cycle constant used in the ring buffer boundary contract verified against the OS partition's write cycle.

### Self-regulation

Poll rate adapts to ring depth without OS involvement:

- Ring depth > high-water mark → poll at minimum cycle interval (maximum throughput).
- Ring depth = 0 → back off by a declared number of cycles (reduces power draw, frees memory bus for prefetch).
- Backoff is bounded: the maximum backoff cycle is declared in `system.cap`. The compiler can prove that no packet waits longer than `max_backoff_cycles` before the ring is polled again.

The critical distinction from thread-based backpressure: the system is not *hoping to catch up*. Under underload, the system deliberately takes longer timeslices — and it knows it is doing so. The backoff is a declared, compiler-visible parameter. The system remains in full control of its own timing; it is not deferring responsibility to a thread scheduler and waiting for an interrupt or a wakeup that may or may not arrive promptly. When load returns, the next ring depth check at the cycle boundary triggers the transition back to minimum interval — no signalling, no wakeup latency, no scheduler involvement.

This is the analogue of FPGA flow control: a FIFO's fill level signals the upstream producer to throttle via a `TREADY` signal. The ring depth is the FIFO fill level; the poll backoff is the consumer's self-throttle. In both cases, the system regulates itself from within — it does not request external intervention.

### No packet drop guarantee

Under heavy load the inverse holds: ring depth is high, backoff is zero, and the poll loop runs at minimum cycle interval — the fastest the system can physically drain the ring. Because the ring buffer is pre-allocated at a declared size and the poll rate is at its maximum, packets are never dropped due to scheduling delay, interrupt deferral, or softirq starvation. The only drop condition is ring overflow, which occurs only if the arrival rate exceeds the declared worst-case processing throughput — a condition the compiler has already verified cannot occur within the declared cycle budget. If the budget is tight enough that overflow is theoretically possible, the compiler rejects the annotation at build time. The guarantee is not a runtime promise; it is a compile-time certificate.

---

## Rationale

### Interrupts cannot enter the app partition

ADR-0002 establishes that app-partition cores have `irqaffinity` excluding them from interrupt delivery. This means interrupt-driven network reception is structurally unavailable for the app partition — not merely inconvenient. The decision to poll is not a performance optimisation; it is the only legal mechanism under the partition model.

### Polling at a declared cycle makes the network stack clock-aware

The key property distinguishing this from DPDK or XDP is compiler visibility. DPDK polls; the compiler does not know DPDK polls. A declared poll cycle in `system.cap`, referenced by the ring buffer boundary contract and validated by the `#[timeslice(...)]` checker, gives the compiler a complete picture of when the network interface is accessed, how long processing takes, and whether the processing budget fits within the timeslice.

This makes packet processing provably bounded. The compiler can reject a `#[timeslice(budget_ns = 4)]` annotation on a packet processing function whose worst-case instruction count exceeds the budget — before the function is ever called on real data.

### Self-regulation eliminates busy-wait waste without sacrificing latency bounds

A fixed-cycle busy-poll loop wastes CPU cycles when the ring is empty, keeping the core at 100% utilisation and heating the CPU unnecessarily. The naive alternative — `epoll` or `select` — reintroduces the OS and destroys the latency bound.

Self-regulation on ring depth achieves both goals: zero OS involvement, and backoff proportional to actual load. The backoff is declared and bounded, so the latency guarantee is preserved: a packet will be processed within `max_backoff_cycles + processing_cycles` of its arrival, proven at compile time.

### Zero-copy path

The ring buffer is a shared memory region mapped into the app partition's address space. The NIC DMA writes directly into this region. The app reads directly from this region. There are no intermediate copies, no `sk_buff`, no socket buffers. The only data movement is the NIC's DMA write and the app's cache-line fetch — one of which is performed by the hardware.

This is structurally identical to FPGA streaming: the AXI stream interface delivers data directly into a FIFO in PL fabric. There is no copy, no buffering in PS memory, no kernel involvement. The CPU partition model replicates this path in software.

### Deterministic clock cycles as the unifying solution

Every alternative rejected in this ADR — DPDK, XDP, `io_uring`, Onload — and every tuning knob spread across the Linux ecosystem — `isolcpus`, `nohz_full`, `irqaffinity`, `rcu_nocbs`, `PREEMPT_RT`, `liburcu`, `numactl`, `cpupower` — exists because the Linux kernel was built without a concept of declared access timing. The absence of that concept forces each subsystem to defend itself independently against temporal uncertainty: the scheduler cannot trust that a critical section will complete before preemption, so it adds locks; the network stack cannot trust that a softirq will run promptly, so it adds buffering; the memory allocator cannot trust that a reader has finished, so RCU defers reclamation. Each defence is correct given the absence of timing declarations. Each defence also carries cost.

Adding deterministic clock cycles as a first-class language and compiler concept does not patch these defences one by one. It removes the premise that makes them necessary. When access timing is declared and compiler-verified, the scheduler knows when a critical section ends, the network stack knows when the ring will be drained, and the allocator knows when the last reader has finished. The defences become redundant — not deprecated, not replaced by another runtime mechanism, but provably unnecessary. The result is a kernel and application stack that is simultaneously simpler, faster, and more correct: simpler because the defensive layers are gone, faster because their runtime cost is gone, and more correct because the proof is static rather than probabilistic.

---

## Alternatives Rejected

### DPDK with no changes

DPDK achieves the zero-copy poll-mode goal at runtime. Its limitation is opacity to the compiler: cycle budgets, poll intervals, and ring depths are runtime constants, not compile-time declarations. DPDK is a good prior art reference and a viable production tool; it is not a clock-aware tool. A clock-aware system may run on top of DPDK drivers but declares its own polling contract above the driver layer.

### XDP / eBPF

XDP runs before `sk_buff` allocation and can forward or drop packets with very low latency. It does not eliminate the interrupt-driven activation (the XDP program is called by the NAPI interrupt handler). The OS is still involved. For app-partition cores, this is not an option for the same reason interrupt-driven reception is not an option: interrupt delivery to those cores is structurally prohibited.

XDP is a good tool for the OS partition — for filtering traffic before it reaches the ring buffer. It is not the right tool for the app partition's internal receive loop.

### `io_uring` with polling mode

`io_uring` in `IORING_SETUP_SQPOLL` mode performs kernel-side polling without requiring a syscall per I/O operation. It reduces the syscall cost but does not eliminate kernel involvement: the submission and completion queues are managed by a kernel thread. Latency is lower than `recvmsg()` but not bounded in the way poll-mode with a declared cycle is bounded. Also requires kernel version ≥ 5.1; the partition model targets any kernel with `nohz_full` support.

### Solarflare / Onload kernel bypass

Onload is a commercial kernel-bypass stack that provides a POSIX socket interface while bypassing the kernel for data delivery. It achieves low latency in practice but is proprietary, tied to specific NIC hardware, and not compiler-visible. Clock-aware programming is hardware-agnostic (any NIC with DMA ring buffer support) and compiler-visible by design.

---

## Consequences

- The app partition's network receive path is a `while true` loop with a declared poll cycle. This loop must be the outermost loop in the app-partition FSM — it is the clock.
- The ring buffer physical address and size in `system.cap` are the NIC's DMA target. NIC configuration (via `ethtool`, `devlink`, or driver-specific tooling) must direct DMA to this address. This configuration is not generated by `cap-gen` in the first version; it is a documented manual step.
- The declared poll cycle in `system.cap` is the minimum granularity for any timeslice annotation in the app partition. A function with `#[timeslice(budget_ns = 2)]` annotated inside a poll loop with a 500-cycle period at 3.6 GHz (≈ 139 ns per period) must complete in 2 ns — the compiler catches violations at build time.
- Maximum packet processing latency is `max_backoff_cycles + worst_case_processing_cycles`, provable at compile time from `system.cap` + the annotated processing function. This bound can be included in a trading system's latency SLA document as a compile-time certificate, not a benchmark median.
