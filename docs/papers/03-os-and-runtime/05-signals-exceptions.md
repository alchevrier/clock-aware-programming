## Signals, Not Exceptions

### There Are No Exceptions

The OOM removal protocol is not a special case. It is an instance of the general model: every error condition in the system is a signal delivered to a circuit via a declared channel. There are no exceptions. There is no stack unwinding. There is no `try`/`catch`. There is no `panic`. There is no `SIGSEGV`. There is no `SIGKILL`.

An exception in a conventional system is what happens when a condition arises that was not declared: a null pointer, a divide-by-zero, an out-of-bounds access, a failed allocation. The language did not declare the condition; the hardware or the OS detects it at runtime; control is transferred by force through an unwinding mechanism to wherever the programmer hoped they remembered to put a handler. The exception is the runtime's way of compensating for a declaration that was never made.

In the clock-aware model, every condition that a circuit can encounter must be declared in its exhaustive match. A condition that the programmer did not handle is a compile error — not a runtime exception. By the time the circuit executes, every reachable state has an explicitly declared handler. There is nothing left for the runtime to intercept.

Error conditions that originate outside the circuit — hardware faults, resource pressure, dependency failures — arrive as signals on declared channels:

| Condition | Signal | Delivered via |
|---|---|---|
| Out of memory | `channel RemovalSignal` | Runtime → circuit |
| Dependency circuit removed | `channel DependencyLost` | Runtime → subscriber circuits |
| Budget overrun detected | `channel BudgetViolation` | Observability sub-circuit → declared handler |
| Hardware fault | `channel HardwareFault` | Driver circuit → subscriber circuits |
| Handoff staleness exceeded | `channel StalenessViolation` | Runtime → declared handler |

Each signal is a typed channel write. The receiving circuit handles it in its next window via its declared exhaustive match — the same mechanism it uses for any other input. There is no separate error-handling path. There is no control-flow transfer outside the circuit's declared windows. The circuit reads the signal, matches it, executes the declared handler, and continues — or acknowledges removal and exits cleanly.

The consequence is that error handling is visible, declared, and compiler-verified in exactly the same way as the happy path. A circuit that subscribes to `channel RemovalSignal` but does not handle it in its match is a compile error. A circuit that handles `DependencyLost` with `ignore` has explicitly declared that it tolerates losing its dependency — and the compiler records that declaration in the manifest. There are no silent failure modes. There are no swallowed exceptions. There are no unhandled panics that terminate the process and leave the system in an unknown state.

A circuit that exceeds its declared budget is not killed. The budget overrun is written to the observability channel. The declared `BudgetViolation` handler runs — which may log the event, reduce the circuit's load, or signal a downstream circuit to shed work. The system adapts through declared channels, not through forced termination.

The signal model unifies what other systems treat as three separate concerns — exceptions, signals, and inter-process communication — into one: a typed channel write, handled by an exhaustive match, at a declared cycle. One mechanism. No exceptions.

An exception is an imperative pattern. It is the runtime seizing control from the programme because the programme did not declare what to do. It belongs to the same family as the scheduler, the GC, the OOM killer, and the lock: compensating machinery that exists because something was never declared. Declare it and the machinery is unnecessary. The clock-aware model does not improve exception handling — it makes the condition that requires it structurally impossible to create.

---

