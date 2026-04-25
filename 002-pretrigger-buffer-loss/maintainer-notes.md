# Maintainer Notes 002

## "The preTriggerBuffers lag in synchStopIdx is designed to prevent exactly this"

Yes and it works in the common case. The lag keeps the consumer behind the producer by `preTriggerBuffers` slots so that when a trigger fires, the pre-trigger slots are not yet within the consumer's processing window.

The failure occurs when `BufferLoop()` wakes and drains its window to the boundary, and `Execute()` fires a trigger on the next cycle marking a slot that was within that window. The lag does not protect against this because the lag is measured from `writeIdx` at the moment `BufferLoop()` captures `synchStopIdx` not from `writeIdx` at the moment the trigger fires. By the time the trigger fires and the retroactive marking happens, `readSynchIdx` may already be past the slot being marked.

## "The consumer will revisit the slot on the next wake"

`readSynchIdx` advances unconditionally:

```
readSynchIdx++;
if (readSynchIdx == numberOfBuffers) { readSynchIdx = 0u; }
```

It is never decremented. `synchStopIdx` on the next wake is computed from the new `writeIdx`, which is always ahead of the slot that was missed. The consumer does not revisit slots behind `readSynchIdx`.

## "This is a theoretical timing issue that does not occur in practice"

The failure requires `BufferLoop()` to be scheduled such that it advances `readSynchIdx` past a slot in the same scheduling window in which `Execute()` subsequently marks that slot. On a multicore system where the consumer thread and the RT thread run concurrently, this is a real interleaving, not a theoretical one. It is more likely when the consumer processes its window quickly relative to the RT cycle period.
