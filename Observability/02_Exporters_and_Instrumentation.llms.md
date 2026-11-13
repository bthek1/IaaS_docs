# Exporters and Instrumentation

Where metrics actually come from: the standard exporters, what each one is for, and how to instrument your own code with the client library.

Author

Benedict Thekkel

## Two Ways To Get A Metrics Endpoint

Prometheus scrapes HTTP endpoints that speak the exposition format. There are only two ways such an endpoint comes to exist.

**An exporter** is a separate process that reads a system it does not own and translates what it finds into metrics. It exists because PostgreSQL, the Linux kernel and a Cisco switch are never going to serve `/metrics` themselves. The exporter is a translation layer, and its quality is mostly a question of which of the underlying system’s numbers it chose to expose.

**Direct instrumentation** is a client library inside the application, incrementing counters in the code paths that matter. This is strictly better where it is possible, because the application knows things no external observer can infer: which business operation a request belonged to, whether a retry was the second or the fifth, why a queue is empty.

The rule is simple. Instrument your own code directly. Use an exporter for everything you did not write.

------------------------------------------------------------------------

## node_exporter: The One To Install First

Machine-level metrics for a Linux or BSD host: CPU, memory, disk, filesystem, network, load, and dozens of optional collectors. If only one exporter runs anywhere, it is this one.

``` yaml
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    command:
      - --path.rootfs=/host
      - --collector.systemd
      - --collector.processes
    pid: host
    network_mode: host
    volumes:
      - /:/host:ro,rslave
    restart: unless-stopped
```

`pid: host` and the read-only root bind are what let a containerised node_exporter see the real machine rather than its own namespace. Without them it reports the container’s view, which is close enough to look plausible and wrong enough to mislead.

### The Metrics That Matter

``` promql
# CPU utilisation as a fraction, per instance
1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))

# Memory in use, excluding cache and buffers
1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)

# Root filesystem usage
1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})

# Disk saturation: fraction of wall time the device was busy
rate(node_disk_io_time_seconds_total[5m])

# Predicted hours until a filesystem fills, from the last 6 hours of trend
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[6h], 3600 * 24) < 0
```

**Use `MemAvailable`, not `MemFree`.** A healthy Linux box has almost no free memory because the kernel uses the rest for page cache, which it will hand back on demand. Alerting on `MemFree` fires constantly on a perfectly healthy machine and is the most common false-positive alert in existence.

**`predict_linear` on disk beats a static threshold.** An 85 percent full disk that has been 85 percent full for a year is not an incident. A disk that will be full in six hours is, even at 60 percent today.

------------------------------------------------------------------------

## The Standard Exporter Set

| Exporter | Port | Scrapes | Notes |
|----|----|----|----|
| `node_exporter` | 9100 | Linux host | The baseline. `windows_exporter` for Windows |
| `cAdvisor` | 8080 | Container CPU, memory, network, per container | Built into the kubelet already; run standalone on a plain Docker host |
| `kube-state-metrics` | 8080 | Kubernetes object state from the API server | Not resource usage. Desired versus actual: replicas, phases, conditions |
| `blackbox_exporter` | 9115 | Probes endpoints from outside: HTTP, TCP, ICMP, DNS, TLS expiry | The only one in this list that is a synthetic check |
| `postgres_exporter` | 9187 | Connections, locks, replication lag, per-table stats |  |
| `mysqld_exporter` | 9104 | The same for MySQL and MariaDB |  |
| `redis_exporter` | 9121 | Keyspace, memory, evictions, replication |  |
| `nginx-prometheus-exporter` | 9113 | Requests, connections, upstream status | Needs the stub_status or Plus API |
| `DCGM exporter` | 9400 | NVIDIA GPU utilisation, memory, temperature, power, ECC | The one that matters on a GPU box |
| `smartctl_exporter` | 9633 | Disk SMART attributes, reallocated sectors, wear | Predicts the failure node_exporter reports after the fact |
| `snmp_exporter` | 9116 | Switches, routers, UPSes, anything that only speaks SNMP | Config generated from MIBs |
| `Pushgateway` | 9091 | A holding pen for short-lived batch jobs to push to | Use sparingly, see below |

### cAdvisor Versus kube-state-metrics

These are constantly confused and they answer opposite questions.

**cAdvisor reports what containers are doing**: this pod is using 1.4 cores and 2 GB. It is a resource-usage view, read from the cgroup filesystem.

**kube-state-metrics reports what Kubernetes thinks should be true**: this deployment wants 3 replicas and has 2 ready, this pod has been `Pending` for 11 minutes, this node is `NotReady`, this job failed. It never looks at a container. It reads the API server and turns object state into metrics.

Alerting on a crash-looping pod needs kube-state-metrics. Alerting on a memory-hungry pod needs cAdvisor. A cluster needs both, and kube-state-metrics is the largest single source of cardinality in most clusters, which is why the `kube_pod_labels` drop rule appears in so many configs.

### blackbox_exporter Works Differently

Every other exporter is scraped for its own metrics. blackbox_exporter is scraped with a **parameter** telling it what to probe, and it performs the probe during the scrape and reports the result.

``` yaml
  # blackbox.yml, the exporter's own config: named probe modules
  modules:
    http_2xx:
      prober: http
      timeout: 5s
      http:
        valid_status_codes: [200]
        follow_redirects: true
        preferred_ip_protocol: ip4
```

``` yaml
  # prometheus.yml: the relabelling dance that makes it work
  - job_name: blackbox-http
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://bthek1.github.io/IaaS_docs/
          - http://192.168.2.205:8888/
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target      # the URL becomes the ?target= param
      - source_labels: [__param_target]
        target_label: instance            # and also the instance label
      - target_label: __address__
        replacement: blackbox-exporter:9115   # scrape the exporter, not the URL
```

That three-rule block is the canonical relabelling example and worth understanding line by line, because it is the clearest case of `relabel_configs` rewriting who gets scraped. The target list holds URLs to probe; the rules move each URL into a query parameter and then point the actual scrape at the exporter.

Useful results: `probe_success`, `probe_duration_seconds`, `probe_http_status_code`, and `probe_ssl_earliest_cert_expiry`, which is how a TLS certificate expiry alert is built.

### Pushgateway Is Usually The Wrong Answer

It exists because a batch job that runs for 30 seconds cannot be scraped. It accepts pushed metrics and holds them for Prometheus to scrape later.

The problems: pushed values persist forever until explicitly deleted, so a decommissioned job keeps reporting its last value indefinitely; the `up` health signal is lost, because Pushgateway is always up whether or not the job ran; and it becomes a single point of failure shared by everything that pushes to it.

Prefer alerting on a `_last_success_timestamp_seconds` gauge going stale, or on the job’s effect rather than the job itself. Where a push really is needed, the OTel Collector is usually a better home for it than Pushgateway.

------------------------------------------------------------------------

## Instrumenting Your Own Code

`prometheus_client` is the Python library. It holds a registry, exposes it over HTTP, and gives you the four metric types.

``` python
from prometheus_client import Counter, Gauge, Histogram, start_http_server

REQUESTS = Counter(
    "app_requests_total",
    "Total requests handled.",
    ["method", "route", "status"],
)
IN_FLIGHT = Gauge(
    "app_requests_in_flight",
    "Requests currently being handled.",
)
LATENCY = Histogram(
    "app_request_duration_seconds",
    "Request duration in seconds.",
    ["route"],
    buckets=(0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0),
)

def handle(method, route):
    with IN_FLIGHT.track_inprogress(), LATENCY.labels(route=route).time():
        status = do_work(method, route)
    REQUESTS.labels(method=method, route=route, status=str(status)).inc()
    return status

start_http_server(8000)   # serves /metrics on :8000
```

`track_inprogress()` and `time()` are context managers, which is the point: the gauge decrements and the histogram observes even when the body raises.

### Choosing Buckets

Default histogram buckets run from 5 ms to 10 s, which suits a typical web request and suits almost nothing else. Buckets are fixed at definition time and cannot be changed retroactively, so the choice matters.

Put the boundaries where the decisions are. If the SLO is 300 ms, there must be a bucket edge at 0.3, because `histogram_quantile` interpolates linearly within a bucket and an estimate that straddles the threshold is worthless. For a job measured in minutes, `buckets=(1, 5, 15, 60, 300, 900)` is right and the defaults would put everything in `+Inf`.

Each bucket is a series, so a histogram with 12 buckets and 3 route values is 36 bucket series plus `_sum` and `_count`. Histograms are the usual reason an application’s series count is higher than expected.

### Framework Integration

Most frameworks have a middleware that does the above generically.

``` python
# Django: prometheus_client via django-prometheus, or a plain middleware
# FastAPI / Starlette:
from prometheus_client import make_asgi_app
app.mount("/metrics", make_asgi_app())
```

**Multiprocess servers need the multiprocess collector.** Gunicorn with four workers means four separate registries, and a scrape hits whichever worker the load balancer picks, so counters appear to jump backwards at random. The fix is `PROMETHEUS_MULTIPROC_DIR` pointing at a shared directory plus `MultiProcessCollector`. Getting this wrong produces metrics that look fine on a single-worker dev box and are nonsense in production.

``` python
import os
from prometheus_client import CollectorRegistry, multiprocess, make_asgi_app

def metrics_app():
    registry = CollectorRegistry()
    multiprocess.MultiProcessCollector(registry)   # reads PROMETHEUS_MULTIPROC_DIR
    return make_asgi_app(registry=registry)
```

------------------------------------------------------------------------

## What To Instrument

The useful default is RED for anything that serves requests and USE for anything that is a resource. Both are covered properly in [SLOs and alerting practice](../Observability/slos_and_alerting_practice.llms.md); the instrumentation consequence is short.

**For a service**: a request counter labelled by route and status, and a latency histogram labelled by route. That is two metrics, and they answer rate, errors and duration between them.

**For a resource** (a queue, a pool, a cache): a gauge for current depth or utilisation, a counter for total work done, and a counter for rejections or evictions.

**For a business process**: a counter at each stage transition. `orders_created_total`, `orders_paid_total`, `orders_shipped_total`. These are the metrics that are still useful a year later, because they survive every rewrite of the code underneath.

### And What Not To

- **No unbounded labels.** No user IDs, request IDs, email addresses, full URLs with query strings, or exception messages. This is the cardinality rule from the [overview](../Observability/observability_overview.llms.md) and it is the one that takes servers down.
- **No timestamps as values**, except deliberately as a `_timestamp_seconds` gauge for staleness checks.
- **No metrics that duplicate a log.** If the answer needs the specific event, it is a log line.
- **No per-instance derived values.** Export the raw counter and let PromQL compute the rate. An application that exposes a pre-computed `requests_per_second` gauge has thrown away the ability to aggregate correctly.

------------------------------------------------------------------------

## Verifying An Exporter

Before wiring it into Prometheus, confirm it works and see what it costs.

``` bash
# Does it respond at all
curl -s localhost:9100/metrics | head -40

# How many series is this endpoint about to add
curl -s localhost:9100/metrics | grep -vc '^#'

# Which metric families are the biggest contributors
curl -s localhost:9100/metrics | grep -v '^#' | sed 's/{.*//' | sort | uniq -c | sort -rn | head -20
```

That last command is the one to run against any new exporter. A postgres_exporter with per-table collectors enabled on a database with 2,000 tables is a five-figure series count from a single target, and it is far cheaper to find that out with `curl` than by watching Prometheus fall over.

------------------------------------------------------------------------

## Where Next

- [Prometheus](../Observability/prometheus.llms.md) for scrape config and the relabelling that filters what these produce.
- [PromQL](../Observability/promql.llms.md) for querying the result.
- [OpenTelemetry](../Observability/opentelemetry.llms.md) for the vendor-neutral alternative to the client library.

------------------------------------------------------------------------

Back to top
