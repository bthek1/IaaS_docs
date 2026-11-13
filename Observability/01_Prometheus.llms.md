# Prometheus

The metrics database at the centre of the stack: its data model, how scraping and service discovery work, relabelling, and what retention actually costs.

Author

Benedict Thekkel

## What Prometheus Is

A single binary holding a time series database, a scraper, a rule evaluator and a query engine. It pulls metrics over HTTP from targets it discovers, stores them locally, evaluates rules against them, and answers PromQL queries. It does not cluster, it does not do long-term storage well, and it deliberately does not try to be a general event store.

Those limits are the design. A Prometheus server is meant to be reliable in isolation, so that when the network is broken and half the platform is down, the thing telling you about it is still running. Each server is independent and needs nothing but local disk. Scale by running more of them, not a bigger one, and federate or remote-write when you need a global view. The [long-term storage](../Observability/long_term_storage.llms.md) page covers what happens past that point.

------------------------------------------------------------------------

## The Data Model

Every series is identified by a metric name plus a set of labels. Internally the name is just a label too, `__name__`, which is why a query can match on it like any other.

    <metric_name>{<label>="<value>", ...} <float64 value> <timestamp ms>

    http_requests_total{method="POST", route="/api/orders", status="500"} 4128 1758585600000

Samples are a float64 and a millisecond timestamp. There are no strings, no structured payloads and no events. A metric that wants to carry a message is the wrong signal and belongs in [Loki](../Observability/loki.llms.md).

### The Four Metric Types

The types exist in the client libraries and in the exposition metadata. The database itself only stores floats, so the type is a contract about how to read the numbers, not a storage difference.

| Type | Semantics | Goes up and down | Query it with |
|----|----|----|----|
| Counter | Cumulative total since process start | No, only up or reset to zero | `rate()`, `increase()` |
| Gauge | A value at a point in time | Yes | Read directly, `avg_over_time()`, `delta()` |
| Histogram | Cumulative counts in configured buckets | No | `histogram_quantile()` over `rate()` |
| Summary | Client-computed quantiles | No | Read directly, cannot aggregate |

**Counters are the important one and the most misread.** A counter’s raw value is meaningless. Nobody cares that a process has served 4,128 requests since it last restarted. What matters is the derivative, which is why almost every useful counter query is wrapped in `rate()`. Counters reset to zero when the process restarts, and the query functions are built to detect and correct for that. Details are in [PromQL](../Observability/promql.llms.md).

**Histograms bucket observations at collection time.** A histogram named `http_request_duration_seconds` exposes `_bucket` series with an `le` label, plus `_sum` and `_count`. Quantiles are computed at query time from the buckets, which means they aggregate correctly across instances, and also that the answer is only as precise as the bucket boundaries chosen in code. Native histograms, stable since Prometheus 3.0, replace the fixed buckets with an exponential scheme that is both cheaper and far more precise, and are worth adopting for new instrumentation.

**Summaries compute quantiles in the client.** A p99 from a summary cannot be averaged across ten instances, because a quantile is not a linear function. Ten instances reporting p99 of 200 ms do not mean the fleet p99 is 200 ms. Prefer histograms in nearly every case.

### Naming

The conventions are not cosmetic; tooling and dashboards rely on them.

- Suffix a counter with `_total`.
- Use base units, always: seconds not milliseconds, bytes not megabytes. Convert for display in Grafana.
- Prefix with the subsystem: `http_`, `pg_`, `node_`.
- The name says what is measured; labels say which instance of it. `http_requests_total{route="/api/orders"}` is right, `http_requests_orders_total` is wrong because it cannot be aggregated.

------------------------------------------------------------------------

## The Exposition Format

A target exposes metrics as plain text on an HTTP endpoint, by convention `/metrics`.

    # HELP http_requests_total Total HTTP requests served.
    # TYPE http_requests_total counter
    http_requests_total{method="GET",status="200"} 18432
    http_requests_total{method="GET",status="500"} 17

    # HELP process_resident_memory_bytes Resident memory size in bytes.
    # TYPE process_resident_memory_bytes gauge
    process_resident_memory_bytes 5.3719040e+07

That is the entire protocol. Any process that can serve text over HTTP can be a Prometheus target, which is why the exporter ecosystem is so large. OpenMetrics is the standardised evolution of this format, and Prometheus reads both.

Two consequences worth internalising. The endpoint returns the **current** value, not a history, so the database is built entirely from samples taken at scrape time, and a value that changes twice between scrapes is simply never seen. And the response is generated on request, so an expensive `/metrics` handler is a self-inflicted load problem on a short scrape interval.

------------------------------------------------------------------------

## Scrape Configuration

`prometheus.yml` has three parts that matter: global defaults, rule files, and scrape jobs.

``` yaml
global:
  scrape_interval: 15s          # how often to pull each target
  evaluation_interval: 15s      # how often to evaluate rules
  external_labels:
    cluster: knowledge-lab      # attached on remote_write and federation

rule_files:
  - /etc/prometheus/rules/*.yml

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: node
    static_configs:
      - targets: ["192.168.2.70:9100", "192.168.2.205:9100"]
        labels:
          env: homelab

  - job_name: grafana
    metrics_path: /metrics
    scheme: http
    scrape_interval: 30s        # overrides the global for this job only
    static_configs:
      - targets: ["192.168.2.205:3000"]
```

Every scraped series automatically gets `job` from the job name and `instance` from the target address. Both are added before any relabelling, and both can be rewritten by it.

**Scrape interval is a budget decision, not a taste one.** Halving it doubles the sample count, the disk use and the scrape load on every target. 15 s is the usual default and 30 s or 60 s is fine for slow-moving infrastructure. Going below 10 s should be justified by a specific query that needs it.

**The interval also sets a floor on what `rate()` can see.** A range in a query must span at least two scrapes, which in practice means at least four times the interval to tolerate a missed one. Scraping at 60 s and then writing `rate(x[1m])` produces empty results, and it is a genuinely common mistake.

------------------------------------------------------------------------

## Service Discovery

Static targets do not survive contact with anything dynamic. Prometheus ships discovery mechanisms that produce the target list at runtime, each emitting a set of `__meta_*` labels describing what it found.

| Mechanism | Use |
|----|----|
| `static_configs` | Fixed hosts, home labs, the Prometheus itself |
| `file_sd_configs` | A JSON or YAML file another system writes. The generic escape hatch |
| `kubernetes_sd_configs` | Pods, services, endpoints, nodes, ingresses from the API server |
| `docker_sd_configs` | Containers on a Docker host, labelled from container metadata |
| `consul_sd_configs`, `dns_sd_configs` | Service registries and SRV records |
| `ec2_sd_configs`, `azure_sd_configs`, `gce_sd_configs` | Cloud instance inventories |

`file_sd_configs` deserves particular attention because it turns discovery into somebody else’s problem. Prometheus watches the file and reloads on change, so any script, Ansible run or Terraform output that can write JSON becomes a discovery source.

``` yaml
  - job_name: file-discovered
    file_sd_configs:
      - files: ["/etc/prometheus/targets/*.json"]
        refresh_interval: 30s
```

``` json
[
  {
    "targets": ["192.168.2.205:9100"],
    "labels": {"env": "homelab", "role": "jupyter"}
  }
]
```

------------------------------------------------------------------------

## Relabelling

Relabelling is the rule engine that sits between discovery and storage. It is where most real Prometheus configuration complexity lives, and where most confusion does too, largely because there are two distinct phases with the same syntax.

| Phase | Key | Runs | Use for |
|----|----|----|----|
| Target relabelling | `relabel_configs` | Before the scrape, on the discovered target | Filtering which targets to scrape, rewriting the address, turning discovery metadata into real labels |
| Metric relabelling | `metric_relabel_configs` | After the scrape, on every sample | Dropping expensive series, renaming metrics, stripping labels |

The mental model: `relabel_configs` decides **who** gets scraped, `metric_relabel_configs` decides **what** gets kept.

### The Actions

``` yaml
relabel_configs:
  # keep: scrape only targets whose annotation opts in
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    action: keep
    regex: "true"

  # replace: promote a discovery label to a real one
  - source_labels: [__meta_kubernetes_pod_label_app]
    target_label: app

  # replace with a rewritten address: honour a custom port annotation
  - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
    regex: "([^:]+)(?::\\d+)?;(\\d+)"
    replacement: "$1:$2"
    target_label: __address__

  # labelmap: bulk-copy a family of discovery labels
  - action: labelmap
    regex: __meta_kubernetes_pod_label_(.+)
```

`keep` and `drop` filter on a regex match, `replace` writes a target label, `labelmap` copies a whole family, `hashmod` buckets targets for sharding across several Prometheus servers.

### Dropping Expensive Series

This is the cardinality control valve, and the one to reach for when a single exporter is responsible for most of the series count.

``` yaml
    metric_relabel_configs:
      # kube-state-metrics emits a series per pod label; usually unwanted
      - source_labels: [__name__]
        action: drop
        regex: "kube_pod_labels"

      # drop a whole noisy subsystem
      - source_labels: [__name__]
        action: drop
        regex: "go_gc_.*|go_memstats_.*"

      # strip a high-cardinality label without dropping the metric
      - action: labeldrop
        regex: "pod_template_hash|controller_revision_hash"
```

**Labels beginning with `__` are internal and discarded before storage.** `__address__`, `__scheme__`, `__metrics_path__` and every `__meta_*` label exist only during relabelling. To keep discovery metadata, copy it to a normal label name first. Forgetting this is why a carefully constructed `__meta_kubernetes_*` value silently fails to appear in the query result.

------------------------------------------------------------------------

## Storage And Retention

Prometheus writes to a local TSDB directory structured as a write-ahead log plus immutable two-hour blocks, which a background compactor merges into larger blocks over time.

    data/
      wal/                  # write-ahead log, replayed on restart
      chunks_head/          # the in-memory head block, on disk for crash recovery
      01H8X.../             # a compacted block: chunks, index, meta.json, tombstones

Retention is configured by time, by size, or both, whichever triggers first.

``` bash
prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --storage.tsdb.retention.time=15d \
  --storage.tsdb.retention.size=50GB
```

### Sizing It

The useful approximation is one to two bytes per sample after compression, and Prometheus compresses time series very well because consecutive samples of a slow-moving number are highly redundant.

    bytes ~= series * (retention_seconds / scrape_interval_seconds) * bytes_per_sample

For 100,000 series at 15 s for 15 days, at 1.5 bytes per sample:

    100000 * (15*86400 / 15) * 1.5 = 100000 * 86400 * 1.5 = about 13 GB

Memory is the tighter constraint on a small box. The head block holds roughly the last two to three hours of every active series in RAM, and the rule of thumb is a few kilobytes per active series, so 100,000 series is several hundred megabytes before the query engine has done anything. On a 20 GB machine that also runs JupyterLab and loads models, keep the series count in the low tens of thousands and be deliberate about which exporters are enabled.

**Retention past a few weeks is the wrong problem to solve here.** Raising `retention.time` to a year on a single server works until the disk or the compactor gives out. The real answers are `remote_write` to a store built for it, or Thanos sidecars shipping blocks to object storage. See [long-term storage](../Observability/long_term_storage.llms.md).

------------------------------------------------------------------------

## Running It

A minimal compose service, which is how the [LGTM Stack](../Observability/lgtm_stack.llms.md) page wires it up:

``` yaml
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --storage.tsdb.retention.time=15d
      - --web.enable-lifecycle          # allows POST /-/reload
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./rules:/etc/prometheus/rules:ro
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
    restart: unless-stopped

volumes:
  prometheus-data:
```

`--web.enable-lifecycle` turns on `POST /-/reload`, so a config change applies without a restart and without losing the head block. Reload rather than restart whenever possible.

### Endpoints Worth Knowing

| Path | What it gives |
|----|----|
| `/targets` | Every discovered target, its state, and the scrape error if it failed |
| `/service-discovery` | Discovered targets **before** relabelling, with all `__meta_*` labels. The debugging view |
| `/config` | The running config as parsed, after file substitution |
| `/rules` | Loaded recording and alerting rules with their evaluation state |
| `/tsdb-status` | Head cardinality: the top label names and values by series count |
| `/-/reload` | Reload the config (POST, needs the lifecycle flag) |
| `/-/healthy`, `/-/ready` | Liveness and readiness |

`/service-discovery` is the one to open when a relabelling rule is not doing what it should. It shows the raw discovery labels that the rules are matching against, which is almost always where the mistaken assumption is.

------------------------------------------------------------------------

## Operational Traps

**A scrape that times out produces no data, not an error metric.** The target simply goes `up == 0`. Always alert on `up`, because a silently missing metric otherwise looks identical to a healthy system with nothing happening.

**`scrape_timeout` cannot exceed `scrape_interval`.** Prometheus refuses the config. An exporter that takes 20 s to respond cannot be scraped every 15 s, and the fix is a slower interval for that job rather than a global change.

**Staleness is five minutes by default.** When a target disappears, its series do not end instantly; Prometheus marks them stale and queries stop returning them after a short window. An alert on `absent()` needs a `for` duration long enough to cover this or it will flap.

**Every restart loses the head block unless the WAL replays cleanly**, and WAL replay on a large head can take minutes during which the server answers nothing. This is the practical argument against one enormous Prometheus.

**Cardinality problems arrive as an OOM, not a warning.** `/tsdb-status` is the diagnostic, and it is worth looking at before a problem rather than after. The query `topk(10, count by (__name__)({__name__=~".+"}))` gives the same answer from the query side.

------------------------------------------------------------------------

## Where Next

- [Exporters and instrumentation](../Observability/exporters_and_instrumentation.llms.md) for where the metrics come from in the first place.
- [PromQL](../Observability/promql.llms.md) for reading them back.
- [Alerting](../Observability/alerting.llms.md) for rules and Alertmanager.
- [Long-term storage](../Observability/long_term_storage.llms.md) for what to do when 15 days is not enough.

------------------------------------------------------------------------

Back to top
