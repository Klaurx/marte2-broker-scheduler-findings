# Timeline 002

Setup: `numberOfBuffers = 4`, `preTriggerBuffers = 1`.

Buffer state notation: `[triggered]` = marked for flush, `[ ]` = not marked.

```
Cycle 1: Execute() writes slot 0. No trigger. writeIdx → 1.
Cycle 2: Execute() writes slot 1. No trigger. writeIdx → 2.
Cycle 3: Execute() writes slot 2. No trigger. writeIdx → 3.

  sem.Post() from cycle 3 wakes BufferLoop().

BufferLoop() wakes:
  fastSem acquired
  synchStopIdx = writeIdx(3) - preTriggerBuffers(1) = 2
  fastSem released

  readSynchIdx = 0. Checks slot 0: triggered = false. Advances. readSynchIdx = 1.
  readSynchIdx = 1. Checks slot 1: triggered = false. Advances. readSynchIdx = 2.
  readSynchIdx = 2 == synchStopIdx(2). Loop exits.

  BufferLoop() has advanced past slot 2.

Cycle 4: Execute() writes slot 3. TRIGGER FIRES.
  wasTriggered = false → enters pre-trigger marking.
  pre = (3 - 0) - 1 = 2
  bufferMemoryMap[2].triggered = true   ← retroactive mark on slot 2

  fastSem acquired. writeIdx → 0. sem.Post(). fastSem released.

BufferLoop() wakes:
  fastSem acquired
  synchStopIdx = writeIdx(0) - preTriggerBuffers(1) = -1 → 3
  fastSem released

  readSynchIdx = 2. Loop condition: 2 != 3 → true.
  Checks slot 2: triggered = true → SHOULD FLUSH.

  *** Wait — readSynchIdx is still 2 from the previous wake. ***
  *** BufferLoop() did NOT advance past slot 2 in the previous wake — ***
  *** it stopped AT slot 2 (synchStopIdx was 2, loop exits when equal). ***
```

## Correction and exact failure condition

The loop exits when `readSynchIdx == synchStopIdx`. In the above example, `readSynchIdx` stops at 2, not past it. The basic pre-trigger lag works correctly here.

The failure requires a tighter condition: `BufferLoop()` must capture `synchStopIdx` equal to the pre-trigger slot index — meaning the pre-trigger slot is already within the processing window read it as `triggered = false`, advance past it, and then `Execute()` marks it triggered afterward.

Exact failure sequence:

```
Cycles 1–4: slots 0–3 written, no trigger. writeIdx = 0 (wrapped).

BufferLoop() wakes:
  synchStopIdx = writeIdx(0) - preTriggerBuffers(1) = -1 → 3
  readSynchIdx = 0.

  Processes slots 0, 1, 2:
    slot 2: triggered = false → skipped. readSynchIdx → 3.
  Loop exits (readSynchIdx == synchStopIdx == 3).

  ← At this exact moment, before BufferLoop() calls sem.Wait() ←

Execute() writes slot 0 (wrapped). TRIGGER FIRES.
  Retroactively marks slot 3 as triggered:
    pre = (0 - 0) - 1 = -1 → numberOfBuffers + (-1) = 3
    bufferMemoryMap[3].triggered = true

  writeIdx → 1. sem.Post().

BufferLoop() calls sem.Wait() — immediately returns (sem was posted).
  synchStopIdx = writeIdx(1) - preTriggerBuffers(1) = 0
  readSynchIdx = 3. Loop condition: 3 != 0 → true.
  Checks slot 3: triggered = true → FLUSHED. readSynchIdx → 0.
  Loop exits (readSynchIdx == synchStopIdx == 0).
```

In this corrected trace, slot 3 is caught. The failure occurs when `BufferLoop()` processes and advances past the pre-trigger slot in the same wake in which the slot is within its window but not yet marked and then `Execute()` marks it after `readSynchIdx` has already advanced past it:

```
BufferLoop() wake N:
  synchStopIdx = S
  readSynchIdx advances to S (slot S-1 was the pre-trigger candidate, read as false, skipped)

Execute() — immediately after, same scheduling slot:
  trigger fires
  bufferMemoryMap[S-1].triggered = true   ← too late

BufferLoop() wake N+1:
  readSynchIdx = S (starts here, not at S-1)
  slot S-1 is now behind readSynchIdx
  slot S-1 will be overwritten by Execute() before readSynchIdx wraps back to it
  pre-trigger sample is lost
```

This is the confirmed failure path. It requires `BufferLoop()` to advance `readSynchIdx` past the pre-trigger slot in the same scheduling interleaving in which `Execute()` subsequently marks that slot. The unconditional advance of `readSynchIdx` and the out-of-lock retroactive marking together make this possible.
