# Detailed Diagnosis Workflows

## Table of Contents

1. [Memory Leak Investigation](#memory-leak-investigation)
2. [High CPU Investigation](#high-cpu-investigation)
3. [Deadlock Investigation](#deadlock-investigation)
4. [Crash Investigation](#crash-investigation)
5. [Performance Regression](#performance-regression)

---

## Memory Leak Investigation

### Symptoms

- Memory usage grows continuously
- `OutOfMemoryException`
- GC pauses increasing
- Gen 2 collections increasing

### Workflow

```
┌─────────────────────┐
│ 1. Confirm Growth   │
│ dotnet-counters     │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ 2. Take Baseline    │
│ dotnet-gcdump       │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ 3. Wait for Growth  │
│ (5-10 minutes)      │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ 4. Take Second Dump │
│ dotnet-gcdump       │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ 5. Compare Dumps    │
│ Find growing types  │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ 6. Find GC Roots    │
│ dotnet-dump gcroot  │
└─────────────────────┘
```

### Commands

```bash
# Step 1: Monitor memory
dotnet-counters monitor -p <PID> --counters \
  "System.Runtime[gc-heap-size,gen-0-gc-count,gen-1-gc-count,gen-2-gc-count]"

# Step 2 & 4: Collect dumps
dotnet-gcdump collect -p <PID> -o dump1.gcdump
# wait...
dotnet-gcdump collect -p <PID> -o dump2.gcdump

# Step 5: Compare (in dotnet-dump)
dotnet-dump analyze dump2.gcdump
> dumpheap -stat
# Note types with high counts

# Step 6: Find what's holding reference
> dumpheap -type MyLeakingClass
> gcroot <address-from-above>
```

### Common Causes

| Cause | Solution |
|-------|----------|
| Event handlers | Unsubscribe in Dispose |
| Static collections | Use bounded or WeakReference |
| Timers not disposed | Dispose in finalizer |
| HttpClient per request | Use IHttpClientFactory |
| Closures capturing | Avoid capturing large objects |

---

## High CPU Investigation

### Symptoms

- CPU usage > 80% sustained
- Slow response times
- Fan spinning constantly

### Workflow

```
┌─────────────────────┐
│ 1. Confirm High CPU │
│ dotnet-counters     │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ 2. Collect Trace    │
│ dotnet-trace        │
│ 30-60 seconds       │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ 3. Convert Format   │
│ speedscope/PerfView │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ 4. Find Hot Paths   │
│ Look at flame graph │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ 5. Optimize Code    │
│ Focus on top %      │
└─────────────────────┘
```

### Commands

```bash
# Step 1: Monitor CPU
dotnet-counters monitor -p <PID> --counters System.Runtime[cpu-usage]

# Step 2: Collect CPU trace
dotnet-trace collect -p <PID> --profile cpu-sampling --duration 00:00:30

# Step 3: Convert to speedscope
dotnet-trace convert trace.nettrace --format speedscope

# Step 4: Open in browser
# Go to https://speedscope.app and load the JSON file
```

### Common Causes

| Cause | Solution |
|-------|----------|
| Busy loop | Add await/delay |
| Regex without timeout | Add Regex timeout |
| Sync over async | Go async all the way |
| N+1 queries | Eager loading, batching |
| JSON serialization | Use source generators |

---

## Deadlock Investigation

### Symptoms

- Application hangs
- No CPU usage but no progress
- Requests timing out

### Workflow

```
┌─────────────────────┐
│ 1. Get Thread Stacks│
│ dotnet-stack        │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ 2. Find Blocked     │
│ Threads waiting     │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ 3. Identify Lock    │
│ What are they       │
│ waiting for?        │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ 4. Find Cycle       │
│ A waits B waits A   │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ 5. Break Cycle      │
│ Reorder locks       │
└─────────────────────┘
```

### Commands

```bash
# Get all thread stacks
dotnet-stack report -p <PID>

# Or with dotnet-dump
dotnet-dump collect -p <PID>
dotnet-dump analyze <dump-file>
> clrthreads
> setthread <thread-id>
> clrstack
```

### Common Patterns

**Classic Lock Deadlock:**

```csharp
// Thread 1: lock A, then lock B
// Thread 2: lock B, then lock A
// DEADLOCK!

// Fix: Always acquire locks in same order
lock (_lockA)
{
    lock (_lockB)
    {
        // work
    }
}
```

**Async/Await Deadlock:**

```csharp
// ❌ Deadlock in ASP.NET (pre-.NET Core)
public string GetData()
{
    return GetDataAsync().Result; // Blocks waiting for context
}

// ✅ Fix
public async Task<string> GetDataAsync()
{
    return await httpClient.GetStringAsync(url);
}
```

---

## Crash Investigation

### Symptoms

- Application terminates unexpectedly
- `Process crashed` in logs
- Unhandled exception

### Workflow

```
┌─────────────────────┐
│ 1. Enable Dumps     │
│ DOTNET_DbgEnable... │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ 2. Reproduce Crash  │
│ Trigger the issue   │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ 3. Analyze Dump     │
│ dotnet-dump         │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ 4. Get Exception    │
│ printexception      │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ 5. Get Stack Trace  │
│ clrstack            │
└─────────────────────┘
```

### Commands

```bash
# Enable crash dumps (Windows)
set DOTNET_DbgEnableMiniDump=1
set DOTNET_DbgMiniDumpType=4
set DOTNET_DbgMiniDumpName=C:\dumps\dump.dmp

# Enable crash dumps (Linux)
export DOTNET_DbgEnableMiniDump=1
export DOTNET_DbgMiniDumpType=4
export DOTNET_DbgMiniDumpName=/tmp/dump.dmp

# Analyze
dotnet-dump analyze <dump-file>
> pe                  # Print exception
> clrstack            # Current thread stack
> clrthreads          # All threads
> setthread <id>      # Switch thread
```

---

## Performance Regression

### Workflow

```
┌─────────────────────┐
│ 1. Establish        │
│ Baseline Metrics    │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ 2. Run Benchmark    │
│ Before/After        │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ 3. Compare Traces   │
│ What changed?       │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ 4. Profile Hot Path │
│ Focus on diff       │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│ 5. Optimize & Verify│
│ Re-run benchmark    │
└─────────────────────┘
```

### BenchmarkDotNet Setup

```csharp
[MemoryDiagnoser]
[SimpleJob(RuntimeMoniker.Net90)]
public class MyBenchmarks
{
    [Benchmark(Baseline = true)]
    public void OldImplementation()
    {
        // old code
    }

    [Benchmark]
    public void NewImplementation()
    {
        // new code
    }
}

// Run
// dotnet run -c Release
```

### Key Metrics to Compare

| Metric | Tool | Good Value |
|--------|------|------------|
| P99 Latency | Application Insights | < 200ms |
| Allocations | BenchmarkDotNet | Decreasing |
| GC Collections | dotnet-counters | Stable |
| Exception Rate | dotnet-counters | < 1/sec |
