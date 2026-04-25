# Impact 001

## Operational consequence

The real-time thread stalls for the duration of `Synchronise()` on every triggered buffer processed by `FlushAllTriggers()`. If N buffers are triggered, the stall is the sum of N `Synchronise()` call latencies.

For DataSources that write to disk, send over a network, or interact with hardware, `Synchronise()` latency is unbounded from the real-time thread's perspective. Stalls in the range of tens to hundreds of milliseconds are plausible in production deployments.

## Why this is not a performance issue

A performance issue degrades throughput or increases average latency. This is a deterministic deadline violation: the real-time control cycle cannot complete on time because the thread responsible for executing it is blocked by a spinlock held during non-real-time I/O.

In a real-time control system, missing a deadline is a correctness failure, not a performance degradation. The downstream effects depend on the controlled system, but they include stale actuation, missed synchronization signals, and watchdog violations.

## Affected configurations

Any deployment using `MemoryMapAsyncTriggerOutputBroker` that calls `FlushAllTriggers()` while real-time execution is active. The `FlushAllTriggers()` function is part of the public API and its documentation explicitly describes use during acquisition   not only at shutdown.

## Relationship to BufferLoop

The normal asynchronous consumer path (`BufferLoop`) does not have this problem: it releases `fastSem` before calling `Synchronise()`. The fix for `FlushAllTriggers()` follows the same pattern   release the lock before processing, not after.
