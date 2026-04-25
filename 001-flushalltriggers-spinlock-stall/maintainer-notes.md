# Maintainer Notes 001

## "FlushAllTriggers is only called at shutdown, not during live execution"

The function documentation states:

> "Note that this happens asynchronously with respect to the buffer consumer thread, so the higher level application using this should make sure that no data is written to this thread while this method is being called."

This documents concurrent use with the consumer thread as a known condition. It does not restrict the function to shutdown contexts. A caller who follows this guidance   ensuring no new data is written   does not prevent the real-time thread from calling `Execute()`, which is what produces the lock contention.

Additionally, the function is part of the public API with no deprecation or restriction markers. Restricting it to shutdown-only use is a documentation change, not a code fix, and leaves the structural problem in place.

## "The lock is held only briefly"

The lock is held for the entire duration of the while loop over all buffers, including each `Synchronise()` call. Brief lock hold time is a property of a specific DataSource implementation, not a guarantee provided by `FlushAllTriggers()`. The lock scope in `FlushAllTriggers()` is determined by the slowest `Synchronise()` implementation that will ever be used with this broker.

## "The caller should not use FlushAllTriggers during RT execution"

If that is the intent, the function should acquire the lock only for the index read and release it before calling `Synchronise()`, as `BufferLoop()` does. The current implementation acquires the lock and holds it across all I/O   which is structurally incompatible with concurrent real-time execution regardless of what the caller is told. The fix is in the lock scope, not in caller discipline.
