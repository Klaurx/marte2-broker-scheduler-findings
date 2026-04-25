# Impact 003

## Operational consequence

The consumer thread reads a mix of data from two different RT cycles from the same buffer slot. The DataSource receives this mixed data via `Synchronise()` and has no way to detect the corruption it receives a complete buffer of the correct size, with correct-looking values that are a partial overwrite of two consecutive signal states.

No error is reported. `ignoreBufferOverrun` suppresses the overrun report, and there is no separate mechanism for detecting concurrent buffer access.

## Affected configurations

Deployments where `SetIgnoreBufferOverrun(true)` is called. This is a deliberate configuration choice, typically made in systems where the consumer is known to occasionally fall behind and dropped data is acceptable. The expectation in such a configuration is that old data is discarded cleanly not that data currently being read is overwritten mid-copy.

## Distinction from intentional lossy behavior

Intentional lossy behavior would skip a buffer slot — drop an unconsumed frame — and move to the next. What this implementation does under `ignoreBufferOverrun = true` is write into a slot while it is being read. The result is not a dropped frame but a corrupted one: the consumer receives a buffer containing partial data from two different time instants. The difference matters for any consumer that performs integrity checks, timestamps, or sequence analysis on the received data.
