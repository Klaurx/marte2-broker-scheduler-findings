# Maintainer Notes 003

## "ignoreBufferOverrun = true means you accept data loss this is intended"

Accepting data loss means accepting that some frames are not delivered to the consumer. It does not mean accepting that a frame currently being delivered is corrupted mid-delivery. The distinction is between skipping a buffer and overwriting one that is being read.

If the intent is lossy-but-not-corrupt behavior, the correct implementation is to skip the write when `toConsume` is true regardless of `ignoreBufferOverrun`. The flag should control whether the skip is reported as an error, not whether the write is skipped at all.

## "The documentation says 'no error will be triggered' that is what happens"

The documentation describes the observable effect on error reporting. It does not describe the change to memory access behavior. A caller reading "no error will be triggered if there is a buffer overrun" reasonably interprets this as: the system will handle the overrun silently, without crashing or reporting. It does not imply: the system will write into a buffer the consumer is currently reading.

The gap between the documented contract and the actual behavior is what makes this a finding. If the intent is for `ignoreBufferOverrun = true` to permit concurrent access, the documentation should say so explicitly.

## "The consumer clears toConsume after copying, so the race window is small"

The size of the race window does not determine whether a race exists. On a multicore system, the producer and consumer can execute simultaneously. The window is the entire duration of `BufferLoop()`'s copy loop for that slot which scales with the number of signals and the `Synchronise()` latency. On slow DataSources, this window is not small.
