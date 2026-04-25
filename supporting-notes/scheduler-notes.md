# Scheduler Notes

## GAMScheduler vs FastScheduler

`GAMScheduler` creates and destroys a `MultiThreadService` per state transition. Threads are spawned fresh for each state. The transition involves stopping the old thread pool, creating a new one, and starting it. This is slower but straightforward: each state's threads are cleanly separated.

`FastScheduler` maintains a persistent thread pool across state transitions. Threads are mapped to CPU masks at configuration time and persist for the application lifetime. State transitions are handled by changing what `rtThreadInfo[idx]` those threads read, controlled by the `currentStateIdentifier` pointer. Idle threads wait on `unusedThreadsSem`.

The `FastScheduler` design introduces the state transition ordering question (see eliminated-findings.md, Candidate A). The ordering is preserved by the call sequence and semaphore semantics.

## countingSem in FastScheduler

`countingSem` is a `CountingSem` created with `maxNThreads`. With `superFast = 0` (default), `StartNextStateExecution()` calls `countingSem.Reset()` before posting `eventSem`. RT threads call `countingSem.WaitForAll()` after waking from `eventSem`. This ensures all threads complete their previous cycle before any begins the new one — a cycle barrier.

With `superFast = 1` (`NoWait = 1`), this barrier is skipped. Threads from the previous cycle can be executing simultaneously with threads of the next cycle. This is documented as a known consequence of the configuration.

## currentStateIdentifier

Declared in `GAMSchedulerI` as `uint32 *currentStateIdentifier` — a pointer to memory owned by `RealTimeApplication`. `GetIndex()` on the application reads through this pointer. The double-buffer indexing (0 and 1) is the mechanism by which the scheduler selects which `rtThreadInfo` slot and which `ScheduledState` to use.
