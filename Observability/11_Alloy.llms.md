# Grafana Alloy

What Grafana’s Collector distribution adds over upstream: the component graph, the Alloy configuration syntax, clustering, and when the difference actually matters.

Author

Benedict Thekkel

## What Alloy Is

Grafana Alloy is a distribution of the OpenTelemetry Collector. It embeds upstream Collector components and adds Grafana’s own, replaces the YAML pipeline config with a programmable configuration language, and includes the collection features that used to live in Promtail and the Grafana Agent.

It replaces all of them. Grafana Agent (both static and flow modes) and Promtail are deprecated in its favour, with Promtail’s support ending in 2026, which is why so many existing setups are mid-migration.

**Read [the Collector page](../Observability/otel_collector.llms.md) first.** Receivers, processors, exporters, the agent and gateway patterns, batching and backpressure are all the same here. This page is only the delta.

------------------------------------------------------------------------

## The Delta

|  | Upstream Collector | Alloy |
|----|----|----|
| Config | YAML, pipelines as lists | Alloy syntax, a component graph with references |
| Composition | Fixed pipeline order | Any DAG, components wired by expression |
| Logic in config | None | Expressions, loops over discovery results, conditionals |
| Prometheus scraping | Via the `prometheus` receiver, a nested Prometheus config | Native, first-class components |
| Service discovery | Inside the Prometheus receiver | Standalone `discovery.*` components usable by anything |
| Log file collection | `filelog` receiver | Native, the Promtail lineage |
| Profiles | Not in core | `pyroscope.*` components, including eBPF |
| Clustering | External load balancer plus `loadbalancing` exporter | Built-in gossip cluster with target sharding |
| UI | `zpages` | A real web UI at :12345 showing the live graph |

**The honest summary**: if the workload is pure OTLP in, OTLP out, upstream is simpler and has no vendor lean. If the workload mixes Prometheus scraping, log tailing, profiling and OTLP, Alloy is materially better because all four are first-class rather than bolted into one receiver.

------------------------------------------------------------------------

## The Component Graph

Alloy configuration declares components and wires them by reference. There is no pipeline list; the graph is implied by which component’s output another component reads.

``` river
// Discovery produces a list of targets
discovery.docker "containers" {
  host = "unix:///var/run/docker.sock"
}

// Relabelling is its own component, consuming that list
discovery.relabel "containers" {
  targets = discovery.docker.containers.targets

  rule {
    source_labels = ["__meta_docker_container_name"]
    regex         = "/(.*)"
    target_label  = "container"
  }
  rule {
    source_labels = ["__meta_docker_container_label_com_docker_compose_service"]
    target_label  = "service"
  }
}

// Two unrelated components consume the SAME discovery output
prometheus.scrape "containers" {
  targets    = discovery.relabel.containers.output
  forward_to = [prometheus.remote_write.default.receiver]
}

loki.source.docker "containers" {
  host       = "unix:///var/run/docker.sock"
  targets    = discovery.relabel.containers.output
  forward_to = [loki.process.default.receiver]
}
```

That reuse is the actual advantage. In upstream YAML, the Prometheus receiver’s discovery and the filelog receiver’s file matching are separate configurations that have to be kept consistent by hand. Here one `discovery.relabel` block feeds metrics scraping and log collection, so labels cannot drift apart.

**`forward_to` takes a list**, so fanning out to two destinations is adding an element. Components with an input expose it as `.receiver`.

------------------------------------------------------------------------

## Alloy Syntax

It is HCL-like: blocks, attributes, and expressions. Not YAML, and not a general-purpose language.

``` river
// Read from the environment or a file
local.file "token" {
  filename  = "/etc/alloy/tempo_token"
  is_secret = true
}

prometheus.remote_write "default" {
  endpoint {
    url = sys.env("PROM_URL")
    basic_auth {
      username = "alloy"
      password = local.file.token.content
    }
    queue_config {
      capacity             = 10000
      max_samples_per_send = 2000
    }
  }
  external_labels = {
    cluster = "knowledge-lab",
    host    = sys.env("HOSTNAME"),
  }
}

// Declare a reusable subgraph
declare "scrape_node" {
  argument "address" {}
  argument "forward_to" {}

  prometheus.scrape "node" {
    targets    = [{"__address__" = argument.address.value, "job" = "node"}]
    forward_to = argument.forward_to.value
  }
}

scrape_node "lab" {
  address    = "192.168.2.205:9100"
  forward_to = [prometheus.remote_write.default.receiver]
}
```

`declare` is the module system, and it is what makes a large Alloy config maintainable rather than a wall of near-identical blocks. Configuration is also reloadable: `SIGHUP` or a `POST` to `/-/reload` applies changes without dropping state.

------------------------------------------------------------------------

## Component Families

| Prefix | Purpose |
|----|----|
| `discovery.*` | Produce target lists: `kubernetes`, `docker`, `file`, `dns`, `ec2`, `relabel` |
| `prometheus.*` | `scrape`, `relabel`, `remote_write`, `receive_http`, `exporter.*` |
| `loki.*` | `source.file`, `source.docker`, `source.journal`, `process`, `write`, `relabel` |
| `otelcol.*` | The embedded upstream components: `receiver.*`, `processor.*`, `exporter.*`, `connector.*` |
| `pyroscope.*` | `scrape`, `ebpf`, `java`, `write` |
| `local.*`, `remote.*` | Files, HTTP, Vault, and Kubernetes secrets as config inputs |
| `declare`, `import.*` | Modules and reuse |

**`prometheus.exporter.*` is worth knowing about.** Alloy embeds many common exporters, so `prometheus.exporter.unix` is node_exporter running inside Alloy with no separate container, and the same exists for postgres, redis, blackbox and others. One process instead of six.

``` river
prometheus.exporter.unix "host" {
  set_collectors = ["cpu", "diskstats", "filesystem", "loadavg", "meminfo", "netdev"]
}

prometheus.scrape "host" {
  targets    = prometheus.exporter.unix.host.targets
  forward_to = [prometheus.remote_write.default.receiver]
}
```

------------------------------------------------------------------------

## Mixing OTel And Grafana Components

`otelcol.*` components are the upstream ones, and they connect to the native ones through converters.

``` river
otelcol.receiver.otlp "default" {
  grpc { endpoint = "0.0.0.0:4317" }
  http { endpoint = "0.0.0.0:4318" }

  output {
    traces  = [otelcol.processor.batch.default.input]
    metrics = [otelcol.processor.batch.default.input]
    logs    = [otelcol.processor.batch.default.input]
  }
}

otelcol.processor.batch "default" {
  output {
    traces  = [otelcol.exporter.otlp.tempo.input]
    metrics = [otelcol.exporter.prometheus.default.input]   // converter
    logs    = [otelcol.exporter.loki.default.input]         // converter
  }
}

otelcol.exporter.otlp "tempo" {
  client {
    endpoint = "tempo:4317"
    tls { insecure = true }
  }
}

// Converters: OTLP metrics -> Prometheus remote write
otelcol.exporter.prometheus "default" {
  forward_to = [prometheus.remote_write.default.receiver]
}

// Converters: OTLP logs -> Loki push
otelcol.exporter.loki "default" {
  forward_to = [loki.write.default.receiver]
}
```

`otelcol.exporter.prometheus` and `otelcol.exporter.loki` are not exporters in the upstream sense. They are adapters from the OTel data model into Alloy’s native Prometheus and Loki paths, which is how one Alloy instance accepts OTLP from applications and Prometheus scrapes from exporters and sends both onward through the same remote write.

------------------------------------------------------------------------

## Clustering

Alloy instances gossip to form a cluster and shard scrape targets among themselves.

``` bash
alloy run /etc/alloy/config.alloy \
  --cluster.enabled=true \
  --cluster.join-addresses=alloy-0:12345,alloy-1:12345
```

``` river
prometheus.scrape "shared" {
  targets    = discovery.kubernetes.pods.targets
  forward_to = [prometheus.remote_write.default.receiver]
  clustering { enabled = true }
}
```

With `clustering` enabled on a component, each instance scrapes only the targets hashed to it, and the set rebalances when an instance joins or leaves. This solves horizontal scaling of scraping without the usual `hashmod` relabelling by hand, and without duplicate scrapes.

It does **not** solve the tail-sampling problem. Routing all spans of a trace to one instance still needs `otelcol.exporter.loadbalancing` with a trace ID routing key, exactly as upstream.

------------------------------------------------------------------------

## The UI

Alloy serves a web UI on :12345 showing the live component graph: every component, its health, its current config after expression evaluation, and the data flowing between them.

This is a genuine advantage over upstream’s `zpages`. A component that is unhealthy shows why, and the evaluated config view resolves the question of what a `sys.env()` or `local.file` reference actually became at runtime, which is otherwise guesswork.

------------------------------------------------------------------------

## Migrating

``` bash
# Promtail, Grafana Agent, Prometheus and upstream Collector configs all convert
alloy convert --source-format=promtail       --output=config.alloy promtail.yaml
alloy convert --source-format=prometheus     --output=config.alloy prometheus.yml
alloy convert --source-format=otelcol        --output=config.alloy otelcol.yaml
alloy convert --source-format=static         --output=config.alloy agent.yaml

# Validate before restarting anything
alloy fmt config.alloy
alloy run --stability.level=generally-available config.alloy
```

The converter output is correct and verbose, producing a component per config block with no attempt to share discovery. Treat it as a starting point to refactor, since the reuse described above is the main reason to be on Alloy at all.

------------------------------------------------------------------------

## Choosing Between Them

**Use upstream Collector** when the pipeline is OTLP in and OTLP out, when vendor neutrality is a stated requirement, when the deployment is already standardised on it, or when the operator model in Kubernetes matters.

**Use Alloy** when the same agent must scrape Prometheus targets, tail log files, collect profiles and accept OTLP, when the Grafana stack is the destination, when target sharding across a fleet is needed without external coordination, or when the configuration is complex enough that expressions and modules beat repeated YAML.

**Do not use Promtail for anything new.** It is deprecated and feature-frozen. Existing deployments work; new ones should be Alloy.

Both speak OTLP, so this is a reversible decision, which is the entire point of having instrumented with OpenTelemetry in the first place.

------------------------------------------------------------------------

## Where Next

- [The OTel Collector](../Observability/otel_collector.llms.md) for the pipeline concepts underneath all of this.
- [Fluent Bit and Vector](../Observability/log_collectors.llms.md) for the non-Grafana alternatives.
- [Loki](../Observability/loki.llms.md) and [Pyroscope](../Observability/pyroscope.llms.md) for two of the destinations.

------------------------------------------------------------------------

Back to top
