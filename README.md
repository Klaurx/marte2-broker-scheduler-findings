# MARTe2 Broker and Scheduler Findings

Component reviewed: `Source/Core/Scheduler/L5GAMs/`

Files examined:
- `MemoryMapAsyncOutputBroker.cpp` / `.h`
- `MemoryMapAsyncTriggerOutputBroker.cpp` / `.h`
- `CircularBufferThreadInputDataSource.cpp` / `.h`
- `FastScheduler.cpp` / `.h`
- `GAMScheduler.cpp` / `.h`
- `GAMSchedulerI.h`

## Findings

| ID | Title | Affected File | Severity |
|----|-------|---------------|----------|
| 001 | Real-time thread stall due to spinlock held across blocking Synchronise() | `MemoryMapAsyncTriggerOutputBroker.cpp` | High |
| 002 | Pre-trigger sample loss due to unlocked retroactive trigger marking | `MemoryMapAsyncTriggerOutputBroker.cpp` | High |
| 003 | ignoreBufferOverrun suppresses the error but permits concurrent overwrite of an active consumer buffer | `MemoryMapAsyncOutputBroker.cpp` | Medium |

Each finding is independently documented and independently defensible.

## Eliminated Candidates

Two additional candidates were identified and eliminated after adversarial review. See `supporting-notes/eliminated-findings.md`.

## Methodology

All findings were derived from source code only. No assumption was made about behavior not visible in the examined files. Each finding was attacked before being reported candidates that did not survive that process were discarded.

Authors:

Klaurx the divine

Ballad of the dead Slowdive
