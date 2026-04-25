# Eliminated Findings

Two candidates were identified during analysis and eliminated after adversarial review. They are documented here to prevent re-investigation of these paths.

## Candidate A FastScheduler state transition ordering violation

**Claim:** RT threads could read a partially-initialized `rtThreadInfo[nextBuffer]` after a state transition because the write to `currentStateIdentifier` and the `eventSem.Post()` call are not atomic.

**Why it was eliminated:**

`CustomPrepareNextState()` fully populates `rtThreadInfo[nextBuffer]` before returning. It is called from `PrepareNextState()` in the state machine, which completes before `StartNextStateExecution()` is called. `StartNextStateExecution()` then calls `eventSem.Post()`, which releases the waiting RT threads.

On Linux, `EventSem` is implemented over `pthread_cond_signal` / `pthread_cond_wait`, which provides acquire/release ordering. The `Post()` in `StartNextStateExecution()` establishes a happens-before relationship with the `Wait()` return in the RT thread. Because `rtThreadInfo[nextBuffer]` is fully written before `Post()` is called, the RT thread's subsequent read of `rtThreadInfo[idx]` is ordered after the write.

The claim required a missing memory barrier. The POSIX semaphore semantics provide that barrier. The candidate does not survive.

## Candidate B CircularBufferThreadInputDataSource TerminateInputCopy counter underflow

**Claim:** `nBrokerOpPerSignalCounter[signalIdx]--` when the counter is already 0 wraps to `0xFFFFFFFF` (unsigned underflow), potentially corrupting the counter state.

**Why it was eliminated:**

The branch condition checks both `== 0u` and `>= nBrokerOpPerSignal[signalIdx]`:

```
nBrokerOpPerSignalCounter[signalIdx]--;
if ((nBrokerOpPerSignalCounter[signalIdx] == 0u) ||
    (nBrokerOpPerSignalCounter[signalIdx] >= nBrokerOpPerSignal[signalIdx])) {
    nBrokerOpPerSignalCounter[signalIdx] = nBrokerOpPerSignal[signalIdx];
    // ... mark buffers as read
    nBrokerOpPerSignalCounter[signalIdx] = nBrokerOpPerSignal[signalIdx];
}
```

If the counter is at 1 and decrements to 0, the `== 0u` branch fires and resets it correctly. If somehow the counter underflows to `0xFFFFFFFF`, that value is `>= nBrokerOpPerSignal[signalIdx]` (which is a small positive number), so the `>=` branch fires and also resets it. The reset logic handles the underflow case. The candidate does not survive.
