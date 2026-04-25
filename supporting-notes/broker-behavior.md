# Broker Behavior Notes

## MemoryMapAsyncOutputBroker buffer lifecycle

Each buffer slot has one state variable: `toConsume` (bool).

```
toConsume = false  →  slot is free, producer may write
toConsume = true   →  slot is occupied, consumer should read
```

Producer (`Execute()`):
1. Checks `toConsume` on `writeIdx` slot. If true and `ignoreBufferOverrun = false`, returns false.
2. Copies GAM memory into `bufferMemoryMap[writeIdx].mem[n]` for all signals.
3. Sets `toConsume = true`.
4. Acquires `fastSem`, increments `writeIdx`, posts `sem`, releases `fastSem`.

Consumer (`BufferLoop()`):
1. Acquires `fastSem`, captures `synchStopIdx = writeIdx`, releases `fastSem`.
2. Walks from `readSynchIdx` to `synchStopIdx`.
3. For each slot where `toConsume = true`: copies to DataSource, calls `Synchronise()`, sets `toConsume = false`.
4. Advances `readSynchIdx` unconditionally.
5. Waits on `sem`.

The `fastSem` protects `writeIdx` and the `sem.Post()` call only. It does not protect the memory copy operations in either direction.

## MemoryMapAsyncTriggerOutputBroker differences from AsyncOutputBroker

Uses `triggered` (bool) instead of `toConsume`. Semantics are similar but the trigger broker adds:

- Pre-trigger buffer window: consumer stays `preTriggerBuffers` slots behind `writeIdx`.
- Post-trigger counter: after a trigger, the next `postTriggerBuffers` slots are also marked triggered even without a trigger signal.
- `wasTriggered` flag: prevents re-marking pre-trigger slots if consecutive triggers fire.
- `numberOfPreBuffersWritten`: prevents marking pre-trigger slots for buffers that have never been written (startup condition).

The trigger broker does not have an `ignoreBufferOverrun` equivalent. It always reports overrun when `triggered` is true on the current `writeIdx`.

## FastPollingMutexSem spinlock semantics

`FastPollingMutexSem` is a spinlock. Callers busy-wait until the lock is available. There is no sleep, no timeout, and no priority inversion mitigation. On a real-time Linux system, a real-time thread spinning on a `FastPollingMutexSem` held by a non-real-time thread (or a real-time thread doing slow I/O) will consume its CPU allocation spinning until the lock is released.

This is the mechanism by which Finding 001 produces a deterministic stall: the RT thread spins on `fastSem` for as long as `FlushAllTriggers()` holds it across `Synchronise()`.

## EventSem ordering semantics

`EventSem` wraps a platform-specific semaphore. On Linux, this is implemented via `pthread_cond_wait` / `pthread_cond_signal`. POSIX specifies that a signal establishes a happens-before relationship with the corresponding wait return. Any write completed before `sem.Post()` is visible to a thread after `sem.Wait()` returns.

This is the ordering guarantee that eliminates the FastScheduler state transition candidate (see eliminated-findings.md, Candidate A).

## Single-buffer edge case

Both async brokers contain special handling for `numberOfBuffers == 1`:

```
if (numberOfBuffers == 1u) {
    readSynchIdx = 0u;
    synchStopIdx = 1;
}
```

With one buffer, the consumer always processes slot 0 and the stop index is set to 1 (an out-of-range sentinel). This bypasses the normal circular index logic. The pre-trigger lag computation (`writeIdx - preTriggerBuffers`) is also bypassed. A deployment with `numberOfBuffers = 1` and `preTriggerBuffers > 0` would violate the precondition `(preTriggerBuffers + postTriggerBuffers) < numberOfBuffers` and is rejected at initialization so this edge case is not reachable in a valid configuration.
