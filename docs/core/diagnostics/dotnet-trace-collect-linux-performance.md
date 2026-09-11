---
title: "Tutorial: Investigate Linux performance with dotnet-trace"
description: Use dotnet-trace collect-linux to investigate managed, native, runtime, and machine-wide Linux performance.
ms.date: 09/11/2026
ms.topic: tutorial
#Customer intent: As a .NET developer on Linux, I want to collect and analyze a trace that identifies the cause of a performance problem.
ai-usage: ai-assisted
---

# Tutorial: Investigate Linux performance with `dotnet-trace collect-linux`

`dotnet-trace collect-linux` records .NET runtime events together with Linux CPU samples, native call stacks, process activity, scheduling data, and kernel events. Use it on .NET 10 and later versions when you need to correlate application activity with activity across the Linux machine.

This tutorial starts with a short machine-wide CPU trace and then adds focused data for a specific question. Five examples show the observations that support a conclusion about managed CPU, native CPU, external CPU competition, allocation and garbage collection (GC), or ThreadPool starvation.

If `collect-linux` isn't available, use Linux `perf` for kernel-level CPU sampling. Also use `perf` when you need `perf.data`, perf-native analysis, or hardware performance counters. For a `perf` walkthrough, see [Debug high CPU usage](debug-highcpu.md#use-perf).

For complete command syntax and platform requirements, see the [`collect-linux` reference](dotnet-trace.md#dotnet-trace-collect-linux).

## Prerequisites

The tutorial requires:

- Linux that meets the [`collect-linux` prerequisites](dotnet-trace.md#prerequisites).
- The [.NET 10 SDK](https://dotnet.microsoft.com/download/dotnet/10.0).
- The latest [`dotnet-trace`](dotnet-trace.md) global tool.
- Visual Studio 18.12 Canary build 12209.137.main on Windows for the managed CPU and basic GC analysis steps.
- [PerfView 3.2.1 or later](https://github.com/microsoft/perfview/releases/tag/v3.2.1) on Windows for thread-time, blocking, contention, native symbol, and deeper stack analysis.
- The [performance scenarios sample](/samples/dotnet/samples/dotnet-trace-collect-linux-performance-scenarios/).

The `collect-linux` verb uses an updated preview `.nettrace` format. The Visual Studio behavior in this tutorial was verified with build 12209.137.main and the evaluated traces. Later builds might behave differently.

## Choose an analysis tool

Start with Visual Studio for a managed CPU hotspot or a basic GC and allocation overview. Its **CPU Usage** tool provides **Functions**, **Call Tree**, **Modules**, and **Flame Graph** views. In the evaluated GC trace, Visual Studio also reported an **Excessive LOH triggered gen2** insight and useful allocation type counts.

Use PerfView 3.2.1 or later for:

- Thread-time, blocked-time, and wait-stack analysis.
- Contention analysis and managed lock-site identification.
- Linux native and ReadyToRun (R2R) symbol lookup.
- Deeper stack analysis when Visual Studio shows unresolved or unwalkable frames.

In the evaluated Visual Studio build, the ThreadPool and blocking traces exposed only raw events. The lock trace didn't produce a contention insight or identify the managed lock site. Linux native frames remained module names and addresses. The symbol settings displayed generic search locations, and the Microsoft Symbol Server was off by default. Temporarily enabling it and reopening the trace didn't resolve `libcoreclr.so` frames.

For the complete evaluation summary, see [Analyze `collect-linux` traces](dotnet-trace.md#analyze-collect-linux-traces).

## Prepare the sample

From the sample directory, build the application and list its scenarios:

```dotnetcli
dotnet build -c Release
dotnet run -c Release --no-build -- --list
```

Use one terminal to run a workload and another terminal to collect the trace. Each workload prints its process ID, symptom, and duration.

## Collect a short machine-wide trace

Start with the default `dotnet-common,cpu-sampling` profiles when you don't yet know which code consumes CPU. The profiles combine lightweight .NET runtime context with Linux CPU samples and native call stacks.

In the workload terminal, run the managed CPU scenario:

```dotnetcli
dotnet run -c Release --no-build -- cpu-hotspot 45
```

Shortly after the workload starts, collect a 30-second trace:

```dotnetcli
sudo dotnet-trace collect-linux \
  --profile dotnet-common,cpu-sampling \
  --duration 00:00:00:30 \
  --output cpu-hotspot.nettrace
```

Machine-wide collection preserves evidence that a process-only trace can miss:

- CPU that competing processes consume.
- Short-lived child processes.
- Work that occurs before a collector can attach to a process.
- Application, runtime, native-library, and kernel CPU in the same interval.

Keep the initial trace short. Scheduler events and detailed runtime events can increase trace size and collection overhead. Add them only when the first trace or the symptom identifies a specific question.

Transfer the `.nettrace` file to a Windows machine that has Visual Studio or PerfView. `collect-linux` captures native and R2R frames. To resolve those frames during analysis, use PerfView and follow [Get symbols for native runtime frames in PerfView](dotnet-trace.md#get-symbols-for-native-runtime-frames-in-perfview).

## Example: Find a managed CPU hotspot

Open `cpu-hotspot.nettrace` in Visual Studio:

1. Select **CPU Usage**.
1. Open **Functions**.
1. Filter to the sample process and the interval when the workload ran.
1. Sort by **Self CPU** to find the application method that directly consumes CPU.
1. Use **Call Tree** to follow the callers back to the application entry point.

**Observation:** `CpuScenarios.Fibonacci` owns nearly all samples for the sample process. Its callers remain in the application stack.

**Conclusion:** The process is compute-bound in `Fibonacci`. The trace identifies both the expensive method and the application path that invokes it.

High self CPU identifies where the CPU sampled execution. The call tree shows the callers that led to that work. Start with self CPU, and then follow callers to find the application code that can change. Use PerfView **CPU Stacks** when you need deeper grouping or stack analysis.

## Choose focused trace data

Select data based on the next question:

| Question | Data to collect | What the data shows |
| --- | --- | --- |
| What code was executing? | CPU samples | The managed, native, or kernel stack running on each CPU. |
| Why wasn't a thread executing? | Thread-time data | Whether the thread was blocked, ready to run, or running. |
| What runtime work occurred? | .NET runtime events | Allocation, GC, contention, ThreadPool, exception, JIT, and loader activity. |

Use these focused configurations for the remaining examples.

### General CPU and native analysis

```dotnetcli
sudo dotnet-trace collect-linux \
  --profile dotnet-common,cpu-sampling
```

### CPU competition and scheduling

```dotnetcli
sudo dotnet-trace collect-linux \
  --profile thread-time,cpu-sampling
```

### Allocation and GC

```dotnetcli
sudo dotnet-trace collect-linux \
  --profile gc-verbose,cpu-sampling
```

### Blocking, contention, and ThreadPool behavior

```dotnetcli
sudo dotnet-trace collect-linux \
  --profile thread-time,cpu-sampling \
  --clrevents threading+contention+waithandle \
  --clreventlevel Verbose
```

Profiles are presets for different data sources. `cpu-sampling` and `thread-time` configure Linux collection. `gc-verbose` configures the .NET runtime provider.

When you use `--clrevents`, select only kernel profiles such as `cpu-sampling` or `thread-time`. If a selected profile already configures `Microsoft-Windows-DotNETRuntime`, the tool uses the profile configuration and ignores `--clrevents`.

## Example: Follow managed code into native code

Run the native CPU scenario:

```dotnetcli
dotnet run -c Release --no-build -- native-cpu 45
```

Collect the general CPU trace for 30 seconds and open **CPU Stacks** in PerfView. Filter to the sample process, and then follow the stack from the managed P/Invoke method into the C runtime.

If the native frame is unresolved, select the module frames and choose **Lookup Symbols**. .NET native and R2R runtime symbols are available from the Microsoft Symbol Server. For native libraries from other Linux distributions, configure PerfView with a local symbol path that contains the matching distribution symbol files.

**Observation:** Samples cross `CpuScenarios.Memcpy` and accumulate in the native `memcpy` function.

**Conclusion:** The application initiates the work, but the CPU cost occurs in a native memory copy. Optimize the copy frequency or size rather than the managed P/Invoke stub.

## Example: Identify external CPU competition

Run the CPU competition scenario:

```dotnetcli
dotnet run -c Release --no-build -- cpu-competition 45
```

Collect the general machine-wide CPU trace. In PerfView, open **CPU Stacks** and compare all processes before you filter to the target.

**Observation:** Several sample worker processes run `SystemScenarios.CpuWorkerAsync` during the same interval. Together, the workers consume the available CPUs, while the target process receives less CPU than its runnable work requests.

The CPU trace shows overlapping competition. To confirm that the target was ready to run but didn't receive CPU, repeat the workload with the CPU competition and scheduling configuration. Compare the target's running time with its ready-to-run time in the thread-time stacks.

**Conclusion:** Other processes saturate the machine. The target isn't blocked inside the application. A process-only trace would hide the competing workers and could lead to an incorrect application-code diagnosis.

Another process with many samples doesn't prove interference by itself. The explanation requires overlapping CPU use, machine saturation, and target threads that are ready but receive less CPU than expected.

## Example: Correlate allocation, GC, and CPU

Run the allocation and GC scenario:

```dotnetcli
dotnet run -c Release --no-build -- allocation-gc 45
```

Collect the allocation and GC configuration for 30 seconds. Start in Visual Studio to review GC insights and allocation type counts. Then use PerfView for allocation stacks and detailed GC data:

1. Open **GCStats** to review collection frequency, pause time, and heap size.
1. Open **GC Heap Alloc Ignore Free (Coarse Sampling)** to find the dominant allocation types and their callers.
1. Open **CPU Stacks** to determine where the process spends CPU.
1. Select the same workload interval in each view.

**Observation:** `System.String` dominates sampled allocations, collections occur repeatedly during the workload, and CPU stacks lead to the sample's string-building path.

**Conclusion:** Repeated string construction creates allocation churn that drives both application CPU and GC work. The trace identifies where objects are created; it doesn't show that the objects remain reachable as a memory leak.

In the evaluated Visual Studio GC trace, the allocation backtrace for `TempObjectClass` followed `user_events_write`, kernel, and native trace-emission frames without reaching the managed allocation site. Use PerfView when the investigation requires the managed allocation caller.

Correlate the allocation, collection, and CPU intervals. GC activity alone doesn't prove that GC is the original cause because application allocation can drive both the collections and the CPU cost.

## Example: Diagnose sync-over-async ThreadPool starvation

Run the sync-over-async scenario:

```dotnetcli
dotnet run -c Release --no-build -- sync-over-async 45
```

Collect the blocking, contention, and ThreadPool configuration for 30 seconds. Use PerfView for this analysis. In the evaluated Visual Studio build, the ThreadPool and blocking traces exposed raw events but didn't provide thread-time, blocked-time, or wait-stack analysis.

1. Open the thread-time stacks for the sample process.
1. Find ThreadPool worker stacks with high blocked time.
1. Inspect `WaitHandleWait` and ThreadPool events in the **Events** view.
1. Follow the blocked stack to the application method that initiated the wait.

**Observation:** Worker stacks block in `Task` wait methods below `BlockingScenarios.SyncOverAsyncAsync`. ThreadPool events show worker growth while the process uses little CPU.

**Conclusion:** Synchronous waits on asynchronous operations make workers unavailable and cause ThreadPool starvation. Replace the blocking `.Result` call with `await` so the worker can process other work during the delay.

Thread-time values are summed across threads, so total blocked time can exceed the trace's wall-clock duration. Focus on the application stacks that account for the blocked workers.

## Use another artifact when the trace can't answer the question

Choose a different artifact when the remaining question isn't represented in the trace:

| Information you need | Use |
| --- | --- |
| Current deadlock owners and the complete wait cycle | A process dump and [Debug a deadlock](debug-deadlock.md). |
| Reference paths that keep managed objects alive | A process dump and [Debug a memory leak](debug-memory-leak.md). |
| Native allocation ownership | A native memory profiler. |
| Logical async, request, or distributed causality | Activities, distributed tracing, or application instrumentation. |
| Hardware counters or `perf.data` | Linux `perf`. |
| Methods removed from physical stacks by optimization | Source, disassembly, or a diagnostic build. |

Collecting more events of the same type can't recover a relationship that the trace doesn't contain.

## Manage collection overhead

Trace overhead depends on event rate, enabled providers, stack capture, CPU count, and workload behavior. High-volume traces can lose events or affect the application.

- Keep the initial trace short.
- Enable only the detailed events needed for the current question.
- Treat event totals as lower bounds when they disagree with application or operating-system counters.
- Repeat the capture with a narrower configuration when exact counts matter.
- Test production collection procedures and container limits before an incident.

## See also

- [`dotnet-trace` reference](dotnet-trace.md)
- [Debug high CPU usage](debug-highcpu.md)
- [Debug ThreadPool starvation](debug-threadpool-starvation.md)
- [Collect diagnostics in Linux containers](diagnostics-in-containers.md)
