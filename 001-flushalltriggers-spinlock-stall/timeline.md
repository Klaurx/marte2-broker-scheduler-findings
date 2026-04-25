# Timeline  001

The following sequence uses two concurrent execution contexts: the state machine thread calling `FlushAllTriggers()`, and the real-time thread calling `Execute()` on each cycle.

```
State machine thread                        Real-time thread
────────────────────────────────────────    ────────────────────────────────────────
FlushAllTriggers() called

fastSem.FastLock() — acquired
                                            Execute() called
                                            fastSem.FastLock() — SPINNING
                                            (busy-wait begins)

Synchronise() called on buffer[0]
  → I/O in progress
  → may block for ms to hundreds of ms
                                            (still spinning — consuming CPU)

Synchronise() returns
buffer[1] ... buffer[N] processed
fastSem.FastUnLock()
                                            fastSem acquired
                                            writeIdx++
                                            sem.Post()
                                            fastSem.FastUnLock()
                                            Execute() returns

                                            (deadline was due during the stall)
                                            (no missed-deadline detection occurs)
```

The real-time thread has no timeout on `fastSem.FastLock()`. It spins until the lock is released. The scheduler does not monitor for this condition.
