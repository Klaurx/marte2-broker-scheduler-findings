# Analysis   001

## Lock scope in FlushAllTriggers()

```
// FlushAllTriggers()   MemoryMapAsyncTriggerOutputBroker.cpp
ret = (fastSem.FastLock() == ErrorManagement::NoError);  // lock acquired

while ((i < numberOfBuffers) && (ret)) {
    if (bufferMemoryMap[idx].triggered) {
        for (c = 0u; (c < numberOfCopies) && (ret); c++) {
            ret = MemoryOperationsHelper::Copy(...);
        }
        if (ret) {
            ret = dataSourceRef->Synchronise();  // potentially blocking   lock still held
        }
        bufferMemoryMap[idx].triggered = false;
    }
    idx++;
    i++;
}

fastSem.FastUnLock();  // lock released only after all buffers processed
```

`fastSem` is a `FastPollingMutexSem`. It is a spinlock   threads waiting on it busy-wait, consuming CPU continuously.

## Lock contention in Execute()

```
// Execute()   MemoryMapAsyncTriggerOutputBroker.cpp
ret = (fastSem.FastLock() == ErrorManagement::NoError);  // same lock
writeIdx++;
if (writeIdx == numberOfBuffers) {
    writeIdx = 0u;
}
if (ret) {
    posted = true;
    ret = sem.Post();
}
fastSem.FastUnLock();
```

`Execute()` is called on every real-time cycle by the scheduler. It must acquire `fastSem` to advance `writeIdx` and post the event semaphore. These are short operations, but they cannot proceed while `FlushAllTriggers()` holds the lock.

## Why this is structural

The failure requires no specific timing or race. The sequence is:

1. `FlushAllTriggers()` is called (e.g., during a state transition to flush buffered acquisition data).
2. `FlushAllTriggers()` acquires `fastSem`.
3. `FlushAllTriggers()` calls `Synchronise()` on one or more triggered buffers. `Synchronise()` performs I/O.
4. The real-time thread calls `Execute()` and attempts to acquire `fastSem`.
5. `fastSem` is a spinlock. The real-time thread busy-waits for the entire duration of `Synchronise()`.
6. `FlushAllTriggers()` completes and releases `fastSem`.
7. The real-time thread resumes   after a delay bounded only by `Synchronise()` latency.

This is not probabilistic. It occurs on every call to `FlushAllTriggers()` where at least one buffer is triggered and `Synchronise()` takes non-trivial time.

## Why BufferLoop does not have this problem

`BufferLoop()` also calls `Synchronise()`, but it does not hold `fastSem` during that call:

```
// BufferLoop()   correct pattern
if (fastSem.FastLock() == ErrorManagement::NoError) {
    synchStopIdx = static_cast<int32>(writeIdx) - static_cast<int32>(preTriggerBuffers);
}
fastSem.FastUnLock();  // released before any I/O

// ... then processes buffers and calls Synchronise() without holding fastSem
```

`FlushAllTriggers()` does not follow this pattern. It holds the lock across the entire processing loop including all `Synchronise()` calls. This is a structural difference between the two code paths that use the same lock.
