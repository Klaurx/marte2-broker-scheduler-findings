# 002 Pre-trigger sample loss due to unlocked retroactive trigger marking

**Affected file:** `Source/Core/Scheduler/L5GAMs/MemoryMapAsyncTriggerOutputBroker.cpp`

**Affected functions:** `MemoryMapAsyncTriggerOutputBroker::Execute()`, `MemoryMapAsyncTriggerOutputBroker::BufferLoop()`

**Violated guarantee:** When a trigger fires, the configured number of pre-trigger samples preceding it are flushed to the DataSource.

## Summary

When a trigger is detected, `Execute()` retroactively marks earlier buffer slots as triggered to capture the pre-trigger history. This marking happens outside the `fastSem` lock, after `writeIdx` has been advanced and the event semaphore has been posted. `BufferLoop()` captures its stop boundary (`synchStopIdx`) under the lock before the trigger fires, then advances `readSynchIdx` past slots unconditionally   without rechecking whether they were subsequently marked. If `BufferLoop()` passes a pre-trigger slot before `Execute()` marks it, that slot is permanently skipped and the pre-trigger sample is silently lost.

## Severity

High. The lost sample is the acquisition data immediately preceding the trigger event typically the highest-value data in a triggered acquisition. The loss is silent: no error is reported, no flag is set, and the DataSource receives no indication that its pre-trigger window is incomplete.
