# Analysis 002

## Retroactive trigger marking in Execute()

When a trigger is detected, `Execute()` marks prior buffer slots to capture the pre-trigger window:

```
// Execute() — trigger detected path
bufferMemoryMap[writeIdx].triggered = (*static_cast<uint8*>(...) > 0u);
if (bufferMemoryMap[writeIdx].triggered) {
    if (!wasTriggered) {
        wasTriggered = true;
        for (j = 0; j < static_cast<int32>(preTriggerBuffers); j++) {
            int32 pre = (static_cast<int32>(writeIdx) - j) - 1;
            if (pre < 0) {
                pre = static_cast<int32>(numberOfBuffers) + pre;
            }
            if (j < numberOfPreBuffersWritten) {
                bufferMemoryMap[pre].triggered = true;  // retroactive marking
            }
        }
    }
    postTriggerBuffersCounter = postTriggerBuffers;
}

// After marking — lock acquired and writeIdx advanced
ret = (fastSem.FastLock() == ErrorManagement::NoError);
writeIdx++;
...
ret = sem.Post();
fastSem.FastUnLock();
```

The retroactive marking of `bufferMemoryMap[pre].triggered = true` happens before `fastSem` is acquired. The lock is acquired only to advance `writeIdx` and post the semaphore.

## Consumer stop boundary capture in BufferLoop()

```
// BufferLoop()
if (fastSem.FastLock() == ErrorManagement::NoError) {
    bufferLoopExecuting = true;
    synchStopIdx = static_cast<int32>(writeIdx) - static_cast<int32>(preTriggerBuffers);
}
fastSem.FastUnLock();

// ... processes slots from readSynchIdx to synchStopIdx
while ((readSynchIdx != static_cast<uint32>(synchStopIdx)) && (ret)) {
    if (bufferMemoryMap[readSynchIdx].triggered) {
        // copy and Synchronise
        bufferMemoryMap[readSynchIdx].triggered = false;
    }
    readSynchIdx++;  // unconditional advance
    if (readSynchIdx == numberOfBuffers) { readSynchIdx = 0u; }
}
```

`synchStopIdx` is captured once under the lock and then used without the lock for the entire processing loop. `readSynchIdx` advances unconditionally after each slot  whether or not that slot was triggered. Once `BufferLoop()` advances past a slot, it does not return to it.

## The failure condition

The window between `BufferLoop()` capturing `synchStopIdx` and `Execute()` performing the retroactive marking is the race surface. If `BufferLoop()` reads a pre-trigger slot as `triggered = false` and advances past it, and `Execute()` subsequently marks that slot `triggered = true`, the slot is permanently skipped.

The `preTriggerBuffers` lag in `synchStopIdx` is designed to keep the consumer behind the producer. This works when the trigger and the consumer wake are well-separated. It fails when `BufferLoop()` wakes and advances to its stop boundary in the same scheduling window in which `Execute()` subsequently fires the trigger and retroactively marks a slot that `BufferLoop()` already passed.

This is not an exotic timing condition. It occurs whenever:
1. `BufferLoop()` wakes (from a previous `sem.Post()`) and drains slots up to the current `synchStopIdx`.
2. `Execute()` fires a trigger on the immediately following cycle and retroactively marks a slot within the range `BufferLoop()` just processed.

These two events share a scheduling window in any deployment where the producer and consumer run at similar rates.
