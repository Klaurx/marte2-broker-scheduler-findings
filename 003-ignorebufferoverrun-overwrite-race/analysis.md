# Analysis 003

## What the documentation says

From `MemoryMapAsyncOutputBroker.h`:

```
/**
 * @brief Sets if buffer overruns shall be ignored (i.e. the consumer thread
 * is not consuming the data fast enough).
 * @param[in] ignoreBufferOverrunIn if true no error will be triggered if
 * there is a buffer overrun.
 */
void SetIgnoreBufferOverrun(bool ignoreBufferOverrunIn);
```

The documented effect is: no error will be triggered. The documentation describes error suppression. It does not describe any change to memory access behavior or synchronization semantics.

## What the implementation does

```
// Execute() — MemoryMapAsyncOutputBroker.cpp
if (!ignoreBufferOverrun) {
    if (bufferMemoryMap[writeIdx].toConsume) {
        ret = false;  // only path that prevents the write
    }
}
// Copy proceeds regardless when ignoreBufferOverrun = true
for (n = 0u; (n < numberOfCopies) && (ret); n++) {
    if (copyTable != NULL_PTR(MemoryMapBrokerCopyTableEntry*)) {
        ret = MemoryOperationsHelper::Copy(
            bufferMemoryMap[writeIdx].mem[n],
            copyTable[n].gamPointer,
            copyTable[n].copySize);
    }
}
bufferMemoryMap[writeIdx].toConsume = true;
```

When `ignoreBufferOverrun = true`, the only guard against writing into an occupied slot is removed. The write proceeds into `bufferMemoryMap[writeIdx].mem[n]` regardless of whether `toConsume` is true.

## Why toConsume does not protect against this

`BufferLoop()` clears `toConsume` after the copy, not before:

```
// BufferLoop() — MemoryMapAsyncOutputBroker.cpp
if (bufferMemoryMap[readSynchIdx].toConsume) {
    for (c = 0u; (c < numberOfCopies) && (ret); c++) {
        ret = MemoryOperationsHelper::Copy(
            copyTable[c].dataSourcePointer,
            bufferMemoryMap[readSynchIdx].mem[c],   // reading here
            copyTable[c].copySize);
    }
    if (ret) {
        ret = dataSourceRef->Synchronise();
    }
    bufferMemoryMap[readSynchIdx].toConsume = false;  // cleared after copy
}
```

The interval between `BufferLoop()` beginning the copy and clearing `toConsume` is the window during which `Execute()` can write into the same slot. With `ignoreBufferOverrun = false`, this is prevented because `Execute()` checks `toConsume` and returns false. With `ignoreBufferOverrun = true`, the check is bypassed and the concurrent write proceeds.

## Summary of the mismatch

The flag is documented as controlling error reporting. Its actual effect is controlling whether memory safety is enforced. A caller who sets `ignoreBufferOverrun = true` to suppress spurious overrun errors in a lossy-acceptable configuration has no indication from the documentation that this also removes the guard against concurrent buffer access.
