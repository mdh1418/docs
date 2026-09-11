---
title: Diagnose performance issues in .NET applications
description: Choose a .NET performance diagnostic tutorial based on the problem you can reproduce.
ms.date: 09/11/2026
ms.topic: conceptual
#Customer intent: As a .NET developer, I want to choose a tutorial that helps me diagnose the performance problem I observe.
ai-usage: ai-assisted
---

# Diagnose performance issues in .NET applications

Start with the problem that you can reproduce. You don't need to know the exact time range, relevant metrics, or constrained resource before you choose a tutorial. Each tutorial begins with the data collection needed to confirm the symptom and identify the next step.

## Choose a performance tutorial

| Observed problem | Start with | What the tutorial shows |
| --- | --- | --- |
| A process consistently uses more CPU than expected. | [Debug high CPU usage](debug-highcpu.md) | Confirm CPU use with `dotnet-counters`, collect a CPU profile, and identify the expensive application path. On .NET 10 and later Linux systems, use `dotnet-trace collect-linux`. Start analysis in Visual Studio for managed hotspots, and use PerfView for deeper Linux stacks and native symbols. Use `perf` when `collect-linux` isn't available or when you need the perf ecosystem. |
| Managed memory continues to increase instead of reaching a steady state. | [Debug a memory leak](debug-memory-leak.md) | Confirm memory growth, collect a process dump, identify the growing object types, and find the reference path that keeps the objects alive. |
| Requests slow down while the ThreadPool queue and worker count increase. | [Debug ThreadPool starvation](debug-threadpool-starvation.md) | Confirm starvation with counters, inspect repeated stacks, and collect wait events when the blocking operation is intermittent. |
| The application stops responding because threads wait indefinitely for locks that other waiting threads own. | [Debug a deadlock](debug-deadlock.md) | Collect a process dump and inspect threads, synchronization blocks, lock owners, and the wait cycle. |
| A Linux CPU problem might involve native code, the kernel, or competing processes. | [Tutorial: Investigate Linux performance with `dotnet-trace collect-linux`](dotnet-trace-collect-linux-performance.md) | Collect a machine-wide trace and interpret concrete managed CPU, native CPU, CPU competition, allocation and GC, and ThreadPool starvation examples. |

## Use counters when the symptom isn't clear

Use [`dotnet-counters`](dotnet-counters.md) to observe a running process while you reproduce the problem:

```dotnetcli
dotnet-counters monitor -n <process-name> --showDeltas
```

Use the output to select a tutorial:

| Counter observation | Likely starting point |
| --- | --- |
| `dotnet.process.cpu.time` increases rapidly during the problem. | [High CPU](debug-highcpu.md) |
| `dotnet.process.memory.working_set` and the managed heap size continue to grow. | [Memory leak](debug-memory-leak.md) |
| `dotnet.thread_pool.queue.length` and `dotnet.thread_pool.thread.count` increase while work slows down. | [ThreadPool starvation](debug-threadpool-starvation.md) |
| The process stops responding, but counters don't identify a changing resource. | If threads might wait on each other's locks, [collect a dump for deadlock analysis](debug-deadlock.md). |

Counters confirm that a resource or runtime condition changes. They don't identify the application code that causes the change. Continue with the linked tutorial to collect stacks, a trace, or a dump.

## See also

- [.NET diagnostic tools](tools-overview.md)
- [Metrics collection](metrics-collection.md)
- [Dumps](dumps.md)
- [Collect diagnostics in Linux containers](diagnostics-in-containers.md)
