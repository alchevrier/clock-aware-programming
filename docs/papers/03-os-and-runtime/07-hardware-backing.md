## Hardware Backing

### The Runtime Resolves Physical Backing — The Programmer Does Not

The programmer writes `channel NicFrame` and calls `in.read()`. There is no DMA in the standard library. There is no `mmap`, no `ioctl`, no driver call, no interrupt registration. The channel is a channel.

The physical mechanism that backs it — DMA ring, interrupt delivery, shared ring buffer, inter-core message — is determined entirely by the subscription type in `system.cap`. The compiler reads the subscription declaration, resolves the physical backing for that channel name, and emits the correct access pattern. The programmer never wrote hardware code. The hardware was never invisible — it was declared once, in the configuration, and the compiler applied it everywhere.

This is the key distinction from conventional hardware abstraction layers. An HAL hides the hardware behind a generic interface and calls through to the driver at runtime. The clock-aware model declares the hardware in `system.cap` and the compiler proves at build time that the access pattern is correct for that hardware. There is no runtime dispatch. There is no driver. The channel type and the subscription declaration are the complete interface.

### The Peripheral Was Already Declaring Its Timing

The deepest insight is that this is not new. Hardware peripherals have always declared their timing — in silicon. The driver writer just was not listening.

Consider a DMA controller with a DREQ (DMA Request) line. The peripheral asserts DREQ exactly when it is ready for a transfer. Power is consumed only when DREQ is asserted, only when the peripheral is ready, only for exactly one transfer unit. Not polling — zero power between transfers. Not interrupt-driven — zero CPU power. Not scheduler-driven — zero context switch. The peripheral declares its timing in hardware. The DMA controller listens. The transfer happens at exactly the rate the hardware needs.

Before driver writers understood this:

```java
while (!peripheralReady()) {}   // polling — burns power
triggerDma();                    // manual, error-prone
while (!dmaDone()) {}           // more polling
```

After understanding DREQ: set the PERMAP register to the peripheral's DREQ number, arm the DMA once, and the hardware paces itself forever. Zero CPU involvement on the hot path. Hundreds of lines of driver code replaced by one 5-bit field in a register — because the peripheral was already declaring its timing. The driver writer just was not listening to it.

On ARM, the runtime can issue `PRFM` (Prefetch Memory) and `RPRFM` (Range Prefetch Memory, SVE2) instructions to pre-warm a circuit's working set before its window opens. During an IDLE tick — when the current core has no circuit assigned — the runtime uses the dispatch table lookahead to identify which circuit fires next and issues a range prefetch for its declared `task`-tier working set. On the hot path, this is redundant: the speculative pre-conditioning mechanism already ensures the working set is in L1 before the window opens. `RPRFM` is most valuable for the **first dispatch of a cold circuit** that has not yet had its working set promoted: the runtime issues the prefetch during the preceding idle window, so the circuit's first execution does not pay a cold-cache penalty. After the first execution, the working set is pinned in L1 and prefetch instructions are unnecessary — the data is already there, declared to be there, proven to be there.

The clock-aware channel model makes this structural. An SPI sensor becomes:

```
// system.cap
channels.SpiReading.device   = SPI0
channels.SpiReading.clock_hz = 1_500_000

// programme
channel SpiReading {
    val element = SensorSample
    val tier    = task
    val size    = 1
}

circuit SpiSensor {
    @Timeslice(core = 1, period = "667ns", budget = "200ns")
    fn read(reading: SpiReading) -> ProcessedReading =
        channel.of(process(reading.get(0)))
}
```

The compiler resolves from the `system.cap` `SpiReading` device declaration: the DREQ line number, the DMA channel, the clock divider from the declared `clock_hz` and `cpu_model`, the CPOL/CPHA mode, the chip-select timing, the bus address translation, the cache flush requirements, and the IRQ registration. The programmer declared what they needed — a 1.5 MHz SPI channel backed by device SPI0. The compiler translated that declaration into every hardware detail the runtime needs. No oscilloscope. No logic analyser. No three days debugging CPOL/CPHA mode mismatch. The device descriptor carries the mode; the compiler applies it; the compiler verifies the element type matches the SPI frame size. If it compiles, the wiring is correct.

### The Runtime Is the Type Enforcer

Hardware-incorrect code does not compile. Not because a linter ran after the fact. Because the type system encodes the hardware model: `channel T` is typed, so reading the wrong type is a type error. Subscription lists are compile-time verified, so reading an undeclared channel is a compile error. Cycle budgets are verified by the proc-macro against `llvm-mca`, so exceeding the budget is a compile error. The type system is the hardware contract. The compiler is the enforcer. The runtime confirms what the compiler already proved.

### Generation 1 to Generation 2 — Closing the Loop

Generation 1 ships a working system: the language, the compiler, the runtime, the kernel circuits — all written in the language — plus ~500 lines of Assembly stubs for boot and hardware initialisation. The Assembly is not hidden; it is the declared boundary between what the language can express and what requires raw instruction sequences.

Generation 2 erases that boundary. The compiler, now running on Generation 1, rewrites the Assembly stubs as declared `channel T` functions. The boot sequence becomes a declared function with a declared timing contract. Hardware initialisation becomes a sequence of channel writes. The ~500 lines of Assembly become zero. The system is fully expressed in the language that proved hardware-correctness of everything else.

A language that can express the kernel can express its own compiler. A compiler that can prove hardware-correctness of timed functions can prove hardware-correctness of itself. The compiler and the kernel become the same artefact: a collection of declared-timing functions, verified and self-validating.

This is what it means for the runtime to be the kernel: not that the kernel is simplified, but that the distinction between compiler and kernel, between language and OS, between proof and execution — collapses.

---

