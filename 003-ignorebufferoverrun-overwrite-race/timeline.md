# Timeline 003

Setup: `numberOfBuffers = 2`, `ignoreBufferOverrun = true`.

```
State: writeIdx = 0, readSynchIdx = 0. bufferMemoryMap[0].toConsume = true.
(Consumer is behind — slot 0 is unconsumed.)

Real-time thread (Execute)              Consumer thread (BufferLoop)
──────────────────────────────          ──────────────────────────────

                                        readSynchIdx = 0
                                        toConsume[0] = true → enter copy
                                        Copy(dataSourcePtr, mem[0], size)
                                        ← copy in progress →

Check toConsume[0]: true
ignoreBufferOverrun = true → skip guard
Copy(mem[0], gamPointer, size)          Copy(dataSourcePtr, mem[0], size)
← writing into mem[0] →                ← reading from mem[0] simultaneously →

                                        Copy completes
                                        Synchronise()
                                        toConsume[0] = false

Copy completes
toConsume[0] = true
fastSem acquired, writeIdx → 1
sem.Post(), fastSem released
```

Both threads access `bufferMemoryMap[0].mem[c]` concurrently without synchronization. The consumer reads a mix of old and new data. No error is reported by either thread.
