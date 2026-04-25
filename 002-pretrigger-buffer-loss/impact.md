# Impact 002

## Operational consequence

Pre-trigger buffers exist to capture system state in the period leading up to a significant event. This is their only purpose. Losing a pre-trigger sample means the acquisition record for a trigger event is silently incomplete.

The loss is not detectable by the DataSource or by any consumer of the data. The DataSource receives fewer `Synchronise()` calls than the pre-trigger window specifies, but it has no way to know this it has no record of what it should have received.

## Affected configurations

Any deployment using `MemoryMapAsyncTriggerOutputBroker` with `preTriggerBuffers > 0`. The failure probability scales with how closely the consumer and producer are phased higher RT frequencies relative to the consumer processing rate increase the likelihood that `BufferLoop()` wakes and advances in the window immediately before a trigger fires.

## Why silence matters

In non-RT systems, data loss of this kind is typically detectable through sequence numbers, checksums, or record counts. `MemoryMapAsyncTriggerOutputBroker` provides none of these. The DataSource receives a `Synchronise()` call for each buffer that was triggered and processed it has no way to verify that the pre-trigger count matches the configured value. The application has no mechanism to detect that this invariant was violated.
