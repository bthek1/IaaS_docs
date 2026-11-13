# Pyroscope

Continuous profiling: the fourth signal, what a flame graph actually shows, how always-on profiling differs from running a profiler, and when it is the only tool that answers the question.

Author

Benedict Thekkel

## The Gap It Fills

Metrics say the service is using 3.2 cores. Traces say 380 ms of the request went into the inventory call. Neither says **which function**.

Profiling has always answered that, but historically as a thing you did: reproduce the problem locally, attach a profiler, read the output. That fails for exactly the problems worth solving, the ones that only appear in production, under real traffic, at 3 a.m., in a process that has since been restarted.

Continuous profiling means every process is sampled continuously, at low overhead, and the profiles are stored like any other telemetry. The question becomes historical: “show me where CPU went in the checkout service between 14:05 and 14:20 yesterday, when the incident happened”. That is a fundamentally different capability from running `py-spy` and hoping to reproduce.

The cost of being always-on is why sampling matters. A sampling profiler interrupts the process on a timer, typically 100 times a second, records the stack, and does nothing else. Overhead is a few percent of CPU at most, which is what makes leaving it on in production defensible.

------------------------------------------------------------------------

## Reading A Flame Graph

The flame graph is the standard rendering and it is routinely misread.

      |=================================== root (100%) ===================================|
      |========== handle_request (62%) ==========|===== background_worker (31%) =====|    |
      |== parse (8%) ==|====== query_db (48%) ===|== flush (28%) ==|                 |    |
                       |== serialize (41%) =====|                                          |

- **Width is the share of samples** in which that function was on the stack. Wider means more time.
- **The y axis is stack depth, not time.** A frame sits on top of its caller. Reading it left to right as a timeline is the classic mistake; the x axis is just an alphabetical or grouped ordering.
- **A wide frame with a wide child** is passing the cost down. A wide frame with **no** wide child is where the time is actually spent, and that is what you are looking for.

The practical reading order: scan the top edge of the graph, not the bottom. The widest plateaus at the top are the leaf functions burning time. Everything below them is just the path that got there.

**Self time versus total time** is the same distinction. Total is the whole subtree; self is the frame alone. A function with 48 percent total and 2 percent self is not the problem, it is calling the problem.

A **differential flame graph** compares two time ranges and colours frames by what grew. This is the single most useful view during an investigation: profile before the deploy, profile after, and the regression is coloured red.

------------------------------------------------------------------------

## What Can Be Profiled

| Profile type | Answers | Available in |
|----|----|----|
| CPU | Which code burns cycles | Everything |
| Alloc / heap | Which code allocates, or holds, memory | Go, Java, Python, .NET, Ruby |
| Goroutines / threads | What is blocked, and where | Go, Java |
| Mutex / block | Which lock is contended | Go, Java |
| Wall clock | Elapsed time including waiting on IO | Python, Ruby, some others |

**CPU and wall-clock profiles answer different questions.** A CPU profile of a service that spends its life waiting on the database shows almost nothing, because waiting does not consume CPU. Wall-clock profiling includes the wait, which is what you want when the complaint is latency rather than cost. Reaching for a CPU profile to explain a slow endpoint is a common and frustrating dead end.

------------------------------------------------------------------------

## Pyroscope

Grafana Pyroscope is the profile store: it ingests profiles, stores them compactly, and serves flame graphs and queries. Grafana renders them natively, including the differential view.

``` yaml
  pyroscope:
    image: grafana/pyroscope:latest
    container_name: pyroscope
    ports:
      - "4040:4040"
    volumes:
      - pyroscope-data:/data
    restart: unless-stopped
```

Profiles are labelled exactly like metrics and selected with a matcher, so everything about cardinality carries across.

    process_cpu:cpu:nanoseconds:cpu:nanoseconds{service_name="api", env="prod"}
     profile type                                labels

### Two Ways To Get Profiles In

**Push, from an SDK in the process.** The application links a small agent that samples itself and ships profiles. It knows the most about itself, so labels can be attached dynamically, including per-request tags.

``` python
import pyroscope

pyroscope.configure(
    application_name="api",
    server_address="http://pyroscope:4040",
    tags={"env": "prod", "region": "ap-southeast-2"},
)

# Tag a section dynamically, so the flame graph can be filtered by it
with pyroscope.tag_wrapper({"endpoint": "/api/orders"}):
    handle_order(request)
```

**Pull, by scraping.** Alloy or the Pyroscope agent scrapes a process’s profiling endpoint on a schedule, exactly like Prometheus scraping `/metrics`. Go’s `net/http/pprof` is the classic case, and eBPF profiling is the version that needs no cooperation from the process at all.

``` river
pyroscope.scrape "default" {
  targets    = [{"__address__" = "api:6060", "service_name" = "api"}]
  forward_to = [pyroscope.write.default.receiver]
  profiling_config {
    profile.process_cpu { enabled = true }
    profile.memory      { enabled = true }
    profile.goroutine   { enabled = true }
  }
}

pyroscope.write "default" {
  endpoint { url = "http://pyroscope:4040" }
}
```

### eBPF Profiling

`pyroscope.ebpf` in Alloy profiles **every process on the host** with no instrumentation, no SDK and no restart, by sampling stacks in the kernel.

``` river
pyroscope.ebpf "system" {
  forward_to = [pyroscope.write.default.receiver]
  targets    = discovery.docker.containers.targets
}
```

The tradeoff is symbolisation. Compiled languages need debug symbols present or the flame graph is full of hex addresses, and interpreted languages need the runtime’s unwinder to be understood, which works well for some and poorly for others. It is the fastest way to get *a* profile of everything, and a worse profile than an in-process SDK gives.

------------------------------------------------------------------------

## Where It Is The Only Answer

**A memory leak.** Heap profiles over time, compared, show which allocation site is growing. Nothing else identifies a leak in a running production process.

**CPU that does not match the work.** Utilisation doubled, request rate did not. A differential profile against last week names the function.

**A regression with no obvious cause.** Deploy, latency up 15 percent, no error rate change, traces show the time spread evenly. That is an in-process cost, invisible to tracing, and a differential profile is the direct route.

**Cost reduction.** Profiling the whole fleet by CPU share turns “the bill is too high” into a ranked list of functions. This is where continuous profiling pays for itself outside incidents, and it is how most organisations end up adopting it.

**What it does not answer**: anything about waiting. A service blocked on a slow dependency has an idle CPU profile. That is a trace question, and reaching for the profiler there wastes an hour.

------------------------------------------------------------------------

## Overhead And Retention

Sampling at 100 Hz costs a low single-digit percentage of CPU, which is the standard default and is acceptable in production. The things that actually hurt are allocation profiling in some runtimes, very deep stacks, and raising the sample rate because higher resolution seemed better.

Storage is modest because profiles compress extremely well: the same stacks repeat. A common retention policy keeps full resolution for a week or two, which covers the investigation window, and that is usually enough. The profile from six months ago describes code that no longer exists.

------------------------------------------------------------------------

## Pyroscope Versus Parca

Both are open-source continuous profiling stores speaking the same pprof format.

**Pyroscope** is the Grafana one, integrates natively into Grafana panels and Explore, has the broadest SDK coverage, and shares the Alloy collection path with the rest of the stack.

**Parca** is CNCF, eBPF-first, focused on zero-instrumentation whole-fleet profiling, with its own UI.

For a stack that is already Grafana, Pyroscope is the obvious pick, purely because the flame graph appears next to the trace and the log rather than in a separate tool.

------------------------------------------------------------------------

## Where Next

- [Tempo](../Observability/tempo.llms.md) for the level above: which service and which span.
- [Alloy](../Observability/alloy.llms.md) for the scrape and eBPF collection config.
- [Grafana](../Observability/grafana.llms.md) for the flame graph panel and the differential view.

------------------------------------------------------------------------

Back to top
