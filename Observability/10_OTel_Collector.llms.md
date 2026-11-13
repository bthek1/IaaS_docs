# OpenTelemetry Collector

The pipeline: receivers, processors, exporters and connectors, the agent and gateway patterns, and the processor ordering that decides whether any of it works.

Author

Benedict Thekkel

## Why A Separate Process

Applications could export telemetry straight to a backend. The reasons almost nobody does:

- **The backend choice stops being a code change.** Switching from Tempo to Jaeger, or adding a second destination, is a config edit in one place rather than a redeploy of forty services.
- **Sampling and filtering need a view the application does not have.** Tail sampling requires seeing a whole trace across services; an SDK only sees its own spans.
- **Retries and buffering belong outside the request path.** An application blocking on a telemetry export during a backend outage is a self-inflicted incident.
- **Credentials live in one place.** No API token in forty deployment manifests.
- **Enrichment happens centrally.** Kubernetes metadata, environment labels and redaction are applied uniformly instead of depending on each service getting it right.

The Collector is that process. It is a single Go binary, configured by YAML, with no state and no dependencies.

------------------------------------------------------------------------

## Distributions

| Distribution | Contains |
|----|----|
| `otelcol` (core) | The minimal, spec-defined set of components |
| `otelcol-contrib` | Core plus the community components. Almost everything useful |
| `otelcol-k8s` | Trimmed for Kubernetes deployments |
| Grafana Alloy | A distribution with its own config language and Grafana-specific components. [Its own page](../Observability/alloy.llms.md) |
| Vendor distributions | ADOT, Splunk, Datadog: core plus their exporters |

**Use `contrib` unless there is a reason not to.** The core distribution lacks the Prometheus receiver’s more useful features, most database receivers, the `k8sattributes` processor and tail sampling, which between them cover most real use cases. The Builder (`ocb`) produces a custom binary with exactly the chosen components, which matters for image size and attack surface in production and not at all in a home lab.

------------------------------------------------------------------------

## The Pipeline Model

Four component types, composed into named pipelines.

      receivers  ->  processors  ->  exporters
                         |
                    connectors (an exporter on one pipeline that is a receiver on another)

``` yaml
receivers:
  otlp:
    protocols:
      grpc: {endpoint: 0.0.0.0:4317}
      http: {endpoint: 0.0.0.0:4318}

  prometheus:
    config:
      scrape_configs:
        - job_name: node
          static_configs:
            - targets: ["node-exporter:9100"]

  filelog:
    include: [/var/log/containers/*.log]
    operators:
      - type: container

processors:
  memory_limiter:
    check_interval: 1s
    limit_percentage: 75
    spike_limit_percentage: 15

  resourcedetection:
    detectors: [env, system, docker]
    override: false

  attributes/redact:
    actions:
      - key: http.request.header.authorization
        action: delete
      - key: user.email
        action: hash

  batch:
    timeout: 5s
    send_batch_size: 8192
    send_batch_max_size: 16384

exporters:
  otlp/tempo:
    endpoint: tempo:4317
    tls: {insecure: true}

  prometheusremotewrite:
    endpoint: http://prometheus:9090/api/v1/write
    target_info: {enabled: true}

  otlphttp/loki:
    endpoint: http://loki:3100/otlp

  debug:
    verbosity: basic

connectors:
  spanmetrics:
    histogram:
      explicit:
        buckets: [10ms, 50ms, 100ms, 250ms, 500ms, 1s, 2s, 5s]
    dimensions:
      - name: http.route
      - name: http.response.status_code

extensions:
  health_check: {endpoint: 0.0.0.0:13133}
  pprof: {endpoint: 0.0.0.0:1777}
  zpages: {endpoint: 0.0.0.0:55679}

service:
  extensions: [health_check, pprof, zpages]
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, resourcedetection, attributes/redact, batch]
      exporters: [otlp/tempo, spanmetrics]

    metrics:
      receivers: [otlp, prometheus, spanmetrics]
      processors: [memory_limiter, resourcedetection, batch]
      exporters: [prometheusremotewrite]

    logs:
      receivers: [otlp, filelog]
      processors: [memory_limiter, resourcedetection, attributes/redact, batch]
      exporters: [otlphttp/loki]

  telemetry:
    metrics:
      level: detailed
    logs:
      level: info
```

**A component defined but not listed in a pipeline does nothing.** This is the most common configuration mistake: adding a processor to the `processors:` block and forgetting to add it to the pipeline’s list. The Collector starts cleanly and silently ignores it.

**The `name/id` syntax creates multiple instances of one component type.** `attributes/redact` and `attributes/enrich` are two independently configured `attributes` processors, and the part after the slash is an arbitrary label.

------------------------------------------------------------------------

## Processor Order Is Semantic

Processors run in the order listed. Two rules follow, and both cause real failures when ignored.

**`memory_limiter` goes first.** It rejects incoming data when memory is over the limit, which is the only thing standing between a backend outage and the Collector being OOM-killed. Placed later, it protects nothing that ran before it.

**`batch` goes last.** Everything before it works on individual items; batching first means later processors operate on batches and lose per-item context. The sole exception is tail sampling, which must see complete traces and therefore comes before batching.

The canonical order:

    memory_limiter -> resourcedetection/k8sattributes -> filter -> transform/attributes -> tail_sampling -> batch

### The Processors Worth Knowing

| Processor | Does |
|----|----|
| `memory_limiter` | Backpressure. Mandatory, first |
| `batch` | Groups items for efficient export. Mandatory, last |
| `resourcedetection` | Adds host, cloud, container metadata from the environment |
| `k8sattributes` | Adds pod, namespace, deployment, node from the API server |
| `attributes` | Insert, update, delete, hash individual attributes |
| `resource` | The same, for resource-level attributes |
| `filter` | Drops whole spans, metrics or logs by an OTTL condition |
| `transform` | Arbitrary OTTL statements. The general-purpose tool |
| `tail_sampling` | Decides on whole traces after buffering them |
| `probabilistic_sampler` | Cheap head sampling in the pipeline |
| `redaction` | Pattern-based removal of sensitive values |

### OTTL

The transform and filter processors use OTTL, a small expression language over the telemetry data model.

``` yaml
processors:
  transform:
    trace_statements:
      - context: span
        statements:
          # Collapse a high-cardinality path into a route template
          - replace_pattern(attributes["url.path"], "/[0-9]+", "/{id}")
          # Promote an attribute onto the resource
          - set(resource.attributes["deployment.environment"], "prod")
          # Mark slow spans for easier querying
          - set(attributes["slow"], true) where end_time_unix_nano - start_time_unix_nano > 1000000000

  filter/drop_health:
    error_mode: ignore
    traces:
      span:
        - 'attributes["http.route"] == "/healthz"'
        - 'attributes["http.route"] == "/metrics"'
    logs:
      log_record:
        - 'IsMatch(body, ".*kube-probe.*")'
```

Dropping health-check spans is nearly always worth doing. They are high volume, perfectly uniform, and they distort every latency percentile downward.

------------------------------------------------------------------------

## Connectors

A connector is an exporter on one pipeline and a receiver on another, which is how one signal is derived from another inside the Collector.

**`spanmetrics`** generates RED metrics from spans: request counts and latency histograms per service and operation. This is the same capability Tempo’s metrics generator provides, done at collection time instead. Doing it in the Collector means it works regardless of the tracing backend; doing it in Tempo means it survives sampling, because Tempo sees what was stored while the Collector sees what was sent.

**`servicegraph`** builds the service dependency graph by matching client and server spans.

**`count`** turns any signal into a count metric, which is a cheap way to get a metric out of logs.

**`forward`** chains pipelines, which is how a shared set of processors is applied before splitting to different destinations.

``` yaml
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp/tempo, spanmetrics]     # spanmetrics consumes the spans
    metrics/from-spans:
      receivers: [spanmetrics]                 # and produces metrics here
      processors: [batch]
      exporters: [prometheusremotewrite]
```

------------------------------------------------------------------------

## Agent And Gateway

Two deployment patterns, usually both at once.

**Agent**: one Collector per host or as a sidecar. It is local, so it can read host files and add host metadata, it collects even when the network to the gateway is down, and the application’s export is a localhost call that cannot fail slowly.

**Gateway**: a horizontally scaled pool receiving from the agents. It is where the expensive, coordinated things happen: tail sampling, aggregation, backend credentials, rate limiting.

      app ---> agent (per host) ---> gateway (pool) ---> Tempo / Prometheus / Loki
               host metadata,        tail sampling,
               file collection       credentials, routing

**Tail sampling forces a constraint on this.** Every span of a trace must reach the same gateway instance, so the agents must use the `loadbalancing` exporter with `routing_key: traceID` rather than a plain round-robin load balancer. Getting this wrong produces partial traces that the sampler judges on incomplete information, and the symptom is traces that are mysteriously missing spans rather than an error.

``` yaml
exporters:
  loadbalancing:
    routing_key: traceID
    protocol:
      otlp:
        tls: {insecure: true}
    resolver:
      dns:
        hostname: otel-gateway-headless.observability.svc
        port: 4317
```

------------------------------------------------------------------------

## Reliability

``` yaml
exporters:
  otlp/tempo:
    endpoint: tempo:4317
    retry_on_failure:
      enabled: true
      initial_interval: 5s
      max_interval: 30s
      max_elapsed_time: 300s
    sending_queue:
      enabled: true
      num_consumers: 10
      queue_size: 5000
      storage: file_storage        # persist the queue across restarts

extensions:
  file_storage:
    directory: /var/lib/otelcol/queue
```

**By default the sending queue is in memory and is lost on restart.** For an agent that can afford to drop telemetry during a backend blip, that is fine. For anything where loss matters, the `file_storage` extension persists it to disk.

**Backpressure propagates.** A full queue plus a full memory limiter means the receiver starts refusing data, and the SDK sees export failures. That is the correct behaviour and is much better than the alternative, which is the Collector consuming all available memory and being killed.

------------------------------------------------------------------------

## Debugging

| Tool | Use |
|----|----|
| `exporters: [debug]` with `verbosity: detailed` | Print everything flowing through to stdout |
| `zpages` on :55679 | `/debug/tracez` and `/debug/pipelinez` show live pipeline state |
| `health_check` on :13133 | Liveness for orchestrators |
| The Collector’s own metrics on :8888 | `otelcol_receiver_accepted_spans`, `otelcol_exporter_send_failed_spans`, `otelcol_processor_dropped_spans` |
| `otelcol validate --config=...` | Catches syntax and component errors without starting |

**Scrape the Collector’s own metrics and alert on them.** `otelcol_exporter_send_failed_spans` and `otelcol_processor_refused_spans` climbing means telemetry is being lost, and nothing else in the system will report that. A silently broken Collector looks exactly like a quiet service.

------------------------------------------------------------------------

## Sizing

The Collector is efficient but not free. A rough starting point is one CPU core per 10,000 to 20,000 spans per second for a simple pipeline, less for one doing heavy OTTL work or tail sampling.

Memory is dominated by the sending queue and by tail sampling’s trace buffer. `num_traces: 100000` with a `decision_wait: 10s` holds ten seconds of every trace in memory, which at any real volume is gigabytes. Size `memory_limiter` below the container limit with enough headroom for a spike, and treat an OOM-killed Collector as a configuration error rather than a resource shortage.

------------------------------------------------------------------------

## Where Next

- [Alloy](../Observability/alloy.llms.md) for Grafana’s distribution of this, and what it adds.
- [Fluent Bit and Vector](../Observability/log_collectors.llms.md) for the log-first alternatives.
- [OpenTelemetry](../Observability/opentelemetry.llms.md) for the SDKs that feed it.

------------------------------------------------------------------------

Back to top
