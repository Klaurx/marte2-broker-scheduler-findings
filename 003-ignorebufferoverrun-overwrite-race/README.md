# 003 ignoreBufferOverrun suppresses the error but permits concurrent overwrite of an active consumer buffer

**Affected file:** `Source/Core/Scheduler/L5GAMs/MemoryMapAsyncOutputBroker.cpp`

**Affected function:** `MemoryMapAsyncOutputBroker::Execute()`

**Violated guarantee:** Signal data written by the real-time thread is not modified while the consumer thread is reading it.

## Summary

`ignoreBufferOverrun` is documented as controlling whether an overrun error is reported. Its implementation goes further: with the flag set to true, `Execute()` skips the `ret = false` guard that would otherwise prevent writing into an occupied buffer slot. Because `toConsume` is cleared by `BufferLoop()` only after the copy from that slot is complete, setting `ignoreBufferOverrun = true` allows `Execute()` to write into a buffer slot while `BufferLoop()` is actively reading from it. The documentation does not disclose this behavior.

## Severity

Medium. The failure requires `ignoreBufferOverrun = true`, which is not the default. When it occurs, it produces silent signal corruption: the consumer reads data that is being partially overwritten by the producer, with no error reported by either side.
