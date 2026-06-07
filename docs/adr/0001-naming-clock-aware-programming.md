# ADR-0001: Naming — Clock-Aware Programming

**Date:** 2026-06-07  
**Status:** Accepted

---

## Context

The concept needed a name before it could be published, referenced, or contributed upstream. The name is permanent on publication — it appears in repository URLs, commit history, and any external citations. Choosing wrong means either renaming (breaking links) or carrying a confusing name indefinitely.

The concept is: a programming model where timing constraints are declared statically and verified by the compiler, analogous to how an FPGA toolchain verifies clock domain crossings at synthesis time.

Four candidates were evaluated.

---

## Decision

**Clock-aware programming.**

---

## Rationale

**"Clock"** implies hardware timing precisely. Anyone who has read a chip datasheet, a Linux kernel timing document, or an HLS constraint file recognises "clock" as the governing synchronisation primitive of deterministic hardware. It is not borrowed or metaphorical — in FPGA design, the clock literally *is* the synchronisation mechanism, and this model transplants that fact to CPUs.

**"Aware"** implies static knowledge, not runtime sensing. The program *knows* its clock domain at compile time. It does not discover timing via a profiler after the fact. The contrast is deliberate: today's systems are clock-*unaware* — they accumulate timing histograms, `rcu_nocbs` tuning, and `nohz_full` boot flags as post-hoc corrections to a model that never encoded timing at all.

**Scales naturally.** Clock-aware kernels. Clock-aware Rust. Clock-aware compilers. Clock-aware data structures. The adjective is not tied to FPGAs, which means it does not require FPGA background to understand.

**Not confusing with existing art.** Real-time systems, RTOS scheduling, and clock synchronisation protocols (PTP, NTP) all use adjacent vocabulary but none use "clock-aware" for this concept.

---

## Alternatives Rejected

### Timeslice programming

**Rejected.** "Timeslice" is OS scheduler vocabulary — it means the quantum of CPU time allocated to a process before preemption. Using it for a model that explicitly *eliminates* the scheduler from the hot path is actively misleading. A reader coming from OS background interprets "timeslice programming" as scheduling-centric, the opposite of the intent.

### Circuit-based programming

**Rejected for the primary name; retained as an analogy.** The FPGA analogy is exact — the app partition behaves like PL fabric, the ring buffer boundary behaves like a clock domain crossing constraint. But "circuit" is insider language. A software engineer without FPGA background reads "circuit-based" and thinks electrical circuits or logic gates, not execution scheduling. The analogy is powerful in explanation but weak as a first-contact label.

### Time-based programming

**Rejected.** Too vague. "Time-based" could mean real-time systems, temporal logic, event sourcing by timestamp, or animations that progress over time. None of those are this. The name does not distinguish the concept from the field it is replacing (real-time systems engineering) or adjacent concepts (event-time processing in streaming systems).

---

## Consequences

- Repository name: `clock-aware-programming` (already created).
- Short alias `cap` is available if needed (implies a hard cycle budget — accurate).
- All ADRs, documentation, and upstream contributions reference "clock-aware" consistently.
- The FPGA analogy is retained in explanations but not in the name.
