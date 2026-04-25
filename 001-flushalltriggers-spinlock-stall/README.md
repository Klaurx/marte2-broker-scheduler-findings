# 001  Real-time thread stall due to spinlock held across blocking Synchronise()

**Affected file:** `Source/Core/Scheduler/L5GAMs/MemoryMapAsyncTriggerOutputBroker.cpp`

**Affected function:** `MemoryMapAsyncTriggerOutputBroker::FlushAllTriggers()`

**Violated guarantee:** Real-time threads must not be blocked by non-real-time I/O operations.

## Summary

`FlushAllTriggers()` acquires `fastSem`   a `FastPollingMutexSem` spinlock   and holds it for the entire duration of iterating all buffers and calling `dataSourceRef->Synchronise()` on each triggered one. The real-time execution path in `Execute()` acquires the same spinlock on every cycle to increment `writeIdx` and post the event semaphore. If `Synchronise()` blocks on slow I/O, the real-time thread spins on the lock for the full duration of that blocking call.

This is a deterministic real-time deadline violation. It is structural: no specific timing is required. Any call to `FlushAllTriggers()` while real-time execution is active produces the stall.

## Severity

High. The stall duration is bounded only by the latency of `Synchronise()` on the underlying DataSource   which may involve disk writes, network sends, or other unbounded I/O. The scheduler has no mechanism to detect or recover from a missed deadline caused by spinlock contention.
