# Tempo

Distributed tracing: what a trace actually is, how Tempo stores them for almost nothing, TraceQL, sampling, and the metrics it can generate from spans.

Author

Benedict Thekkel

## What A Trace Is

A **span** is one unit of work: a name, a start time, a duration, a status, and a bag of attributes. A **trace** is the tree of spans produced by one logical operation, linked by a shared trace ID and by each span recording its parent.

    trace 7f3a...  POST /api/orders                              [========================] 840ms
      |- auth.verify_token                                       [===]                       40ms
      |- db.query  SELECT * FROM users WHERE id = ?              [==]                        25ms
      |- http POST inventory-service/reserve                     [============]             410ms
      |    |- db.query  UPDATE stock SET ...                     [=]                         18ms
      |    |- http POST warehouse-api/allocate                   [=========]                380ms
      |- db.query  INSERT INTO orders ...                        [=]                         30ms
      |- queue.publish  order.created                            [=]                         12ms

That picture is the entire value proposition. The metric said the endpoint takes 840 ms. The trace says 380 ms of it is in a third-party warehouse API two hops away, which no amount of staring at the first service’s dashboards would have revealed.

**Context propagation is what holds it together.** Each service passes the trace ID and its current span ID to the next in a header, conventionally W3C `traceparent`:

    traceparent: 00-7f3a1b2c3d4e5f60718293a4b5c6d7e8-00f067aa0ba902b7-01
                 ver  trace-id (16 bytes)              span-id (8)     flags

A service that drops this header breaks the trace at that point, producing two disconnected fragments. This is the single most common tracing defect, and it is almost always a proxy, a queue, or a hand-rolled HTTP client that does not forward the header. Details are in [OpenTelemetry](../Observability/opentelemetry.llms.md).

------------------------------------------------------------------------

## Tempo’s Design

Tempo indexes almost nothing. Traces are written to object storage keyed by trace ID, with a small per-block index, and that is it. There is no Elasticsearch, no Cassandra, no separate index cluster, which is what older tracing backends needed and what made them hard to run.

The consequence that used to define Tempo: lookup by trace ID was free, and searching for traces by anything else was impossible, so you found the trace ID first, from a log line or an exemplar on a metric graph. TraceQL and the block indexes have since made real search work, but the design instinct remains. **Traces are usually reached from somewhere else**, and the fastest path is still a click from a log line or a latency graph.

      distributor   accepts OTLP, Jaeger, Zipkin. Routes spans by trace ID
           |
      ingester      batches spans per trace, flushes complete-ish traces to blocks
           |
      object store  blocks of traces plus a bloom filter and index per block
           |
      querier       finds the block, fetches the trace
           |
      compactor     merges blocks, enforces retention

``` yaml
  tempo:
    image: grafana/tempo:latest
    command: ["-config.file=/etc/tempo.yaml"]
    volumes:
      - ./tempo.yaml:/etc/tempo.yaml:ro
      - tempo-data:/var/tempo
    ports:
      - "3200:3200"     # tempo http, query API
      - "4317:4317"     # OTLP gRPC
      - "4318:4318"     # OTLP HTTP
```

``` yaml
stream_over_http_enabled: true
server:
  http_listen_port: 3200

distributor:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318

ingester:
  max_block_duration: 5m

compactor:
  compaction:
    block_retention: 336h        # 14 days

storage:
  trace:
    backend: local               # or s3, gcs, azure
    wal:
      path: /var/tempo/wal
    local:
      path: /var/tempo/blocks

metrics_generator:
  registry:
    external_labels:
      source: tempo
  storage:
    path: /var/tempo/generator/wal
    remote_write:
      - url: http://prometheus:9090/api/v1/write
        send_exemplars: true

overrides:
  defaults:
    metrics_generator:
      processors: [service-graphs, span-metrics]
```

------------------------------------------------------------------------

## TraceQL

TraceQL selects **spansets**: groups of spans within a trace that match a condition. The braces hold a span condition, and everything outside them operates on traces.

``` traceql
# Any span from this service
{ resource.service.name = "api" }

# Slow spans on a specific route
{ resource.service.name = "api" && span.http.route = "/api/orders" && duration > 500ms }

# Errors
{ status = error }

# Traces where an api span AND a db span both exist
{ resource.service.name = "api" } && { resource.service.name = "postgres" }

# Traces that touched api but never the cache
{ resource.service.name = "api" } && !{ resource.service.name = "redis" }

# Structural: a db span that is a descendant of a checkout span
{ span.name = "checkout" } >> { span.name =~ "db.*" && duration > 100ms }
```

| Scope | Meaning |
|----|----|
| `resource.` | Attributes of the emitting process: `service.name`, `host.name`, `k8s.pod.name` |
| `span.` | Attributes of the individual span: `http.status_code`, `db.statement` |
| `name`, `status`, `duration`, `kind` | Intrinsics, no prefix |
| `trace:duration`, `trace:rootName` | Trace-level intrinsics |

Structural operators are what make TraceQL more than a filter: `>` for direct child, `>>` for any descendant, `~` for sibling, and their negations. “A slow database call somewhere underneath the checkout handler” is a single expression, and that question is very hard to ask any other way.

### Aggregates

``` traceql
# Traces whose api spans total more than 2 seconds
{ resource.service.name = "api" } | sum(duration) > 2s

# Traces with more than 20 database calls: the N+1 query finder
{ span.name =~ "db.*" } | count() > 20

# Slowest span per trace above a bound
{ resource.service.name = "api" } | max(duration) > 1s
```

`count() > 20` on database spans is the canonical N+1 detector, and it finds in one query what usually takes an afternoon of reading code.

### Metrics From TraceQL

``` traceql
{ resource.service.name = "api" } | rate()
{ resource.service.name = "api" && status = error } | rate() by (span.http.route)
{ resource.service.name = "api" } | quantile_over_time(duration, 0.99) by (span.http.route)
{ resource.service.name = "api" } | histogram_over_time(duration)
```

These run over the trace data directly and return time series that Grafana graphs like any metric. They are expensive over long ranges, since there is no pre-aggregation; for anything permanent, use the metrics generator instead.

------------------------------------------------------------------------

## Sampling

Tracing every request is affordable in a home lab and not at scale. There are two strategies and the difference matters.

**Head sampling** decides at the start of the trace, in the SDK, usually as a fixed probability. It is cheap, it needs no coordination, and the decision propagates in the `traceparent` flags so the whole trace is consistently kept or dropped. Its flaw is that the decision is made before anything interesting has happened, so a 1 percent sample keeps 1 percent of errors too.

``` python
# OTel SDK, head sampling at 10 percent
from opentelemetry.sdk.trace.sampling import TraceIdRatioBased, ParentBased
sampler = ParentBased(root=TraceIdRatioBased(0.1))
```

**Tail sampling** buffers the whole trace and decides after it completes, which allows rules like “keep everything that errored, everything slower than 2 s, and 5 percent of the rest”. It runs in the Collector, not the SDK, and it costs memory plus the constraint that **every span of a trace must reach the same Collector instance**, which needs a load-balancing exporter in front of a Collector pool.

``` yaml
  processors:
    tail_sampling:
      decision_wait: 10s
      num_traces: 100000
      policies:
        - name: errors
          type: status_code
          status_code: {status_codes: [ERROR]}
        - name: slow
          type: latency
          latency: {threshold_ms: 2000}
        - name: baseline
          type: probabilistic
          probabilistic: {sampling_percentage: 5}
```

The usual progression: sample nothing at first, add head sampling when volume hurts, move to tail sampling when the head sample starts hiding the errors you need.

------------------------------------------------------------------------

## The Metrics Generator

Tempo can derive metrics from spans as they arrive and remote-write them to Prometheus. This is the feature that makes tracing pay for itself even when nobody opens a trace.

**`span-metrics`** produces RED metrics for every service and operation without any additional instrumentation: `traces_spanmetrics_calls_total` and `traces_spanmetrics_latency_bucket`, labelled by service, span name, status and whatever dimensions are configured. A service that emits traces gets request rate, error rate and latency histograms for free.

**`service-graphs`** produces `traces_service_graph_request_total` and a duration histogram for each **edge** between services, by matching client spans to server spans. This is where Grafana’s service map comes from, and it is the only automatic source of a real dependency graph, because it is built from observed traffic rather than from a diagram somebody drew.

These write into Prometheus, so they are subject to the usual cardinality rules. Adding `http.route` as a dimension is usually right; adding `http.url` is the familiar mistake.

------------------------------------------------------------------------

## Exemplars: The Link From Metric To Trace

An exemplar attaches a trace ID to a specific sample in a histogram bucket. On a latency graph this renders as a clickable dot: the p99 spiked, here is an actual trace from that spike.

``` python
# prometheus_client supports exemplars on counters and histograms
LATENCY.labels(route=route).observe(
    elapsed,
    exemplar={"trace_id": current_trace_id()},
)
```

Prometheus needs `--enable-feature=exemplar-storage`, and the scrape must use the OpenMetrics format. In Grafana, the Prometheus datasource is configured with an exemplar link pointing at the Tempo datasource and the `trace_id` label.

This closes the loop, and it is what the three signals together are for:

      metric graph shows a p99 spike
            |  exemplar
      one real trace from that spike
            |  span attributes and trace ID
      the log lines emitted during that span

Grafana wires the other two directions too: **trace to logs** (from a span, query Loki for lines carrying that trace ID) and **logs to trace** (from a log line’s trace ID, open the trace). Both are configured on the datasource, not in the panels, and both depend on the trace ID actually being present in the log line, which is the job of the logging integration in the OTel SDK.

------------------------------------------------------------------------

## Tempo Versus Jaeger

|  | Tempo | Jaeger |
|----|----|----|
| Storage | Object storage only | Cassandra, Elasticsearch, or its own badger |
| Search | TraceQL, including structural queries | Tag-based search on an indexed store |
| Ops burden | Low | Higher, inherits the index backend’s |
| UI | Grafana | Its own, plus a Grafana datasource |
| Derived metrics | Built in: span metrics, service graphs | Via the SPM setup with a separate Prometheus |
| Ingest | OTLP, Jaeger, Zipkin | OTLP, Jaeger, Zipkin |

Both consume OTLP, so instrumenting with OpenTelemetry leaves the choice reversible. Tempo fits a Grafana stack and an object-storage cost model; Jaeger is the CNCF standard and is everywhere in existing deployments.

------------------------------------------------------------------------

## Traps

**A broken trace is a propagation problem, not a Tempo problem.** Disconnected fragments mean a hop dropped `traceparent`. Check proxies, message queues, and any HTTP client constructed by hand.

**Span attributes are not free.** They are stored per span, and a `db.statement` attribute carrying a full query on a hot path is a large volume of storage.

**The metrics generator writes into Prometheus’s cardinality budget.** Configure its dimensions with the same care as any other metric.

**Clock skew across hosts distorts the waterfall.** Spans can appear to start before their parent. Tempo displays what it was given; the fix is NTP.

------------------------------------------------------------------------

## Where Next

- [OpenTelemetry](../Observability/opentelemetry.llms.md) for producing the spans and propagating context.
- [The OTel Collector](../Observability/otel_collector.llms.md) for tail sampling and routing.
- [Pyroscope](../Observability/pyroscope.llms.md) for the level below a span: which function inside it burned the time.

------------------------------------------------------------------------

Back to top
