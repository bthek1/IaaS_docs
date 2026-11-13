# Log Collectors: Fluent Bit and Vector

The two serious non-Grafana collection agents, what each is genuinely better at, and the log-pipeline problems all of them share.

Author

Benedict Thekkel

## Why This Page Exists

[Alloy](../Observability/alloy.llms.md) and [the OTel Collector](../Observability/otel_collector.llms.md) are one answer to collection. Fluent Bit and Vector are the other two, and they arrived from the logging side rather than the tracing side, which shows in what they are good at.

All four occupy the same layer and all four can ship logs to Loki. The differences that matter are throughput, transformation power, and how much of the ecosystem assumes you are running one of them.

------------------------------------------------------------------------

## The Field

|  | Fluent Bit | Vector | OTel Collector | Alloy |
|----|----|----|----|----|
| Written in | C | Rust | Go | Go |
| Memory, idle | ~3 MB | ~30 MB | ~50 MB | ~80 MB |
| Origin | Logging (CNCF, Fluentd family) | Logging (Datadog) | Tracing | Metrics and logging |
| Transform language | Lua, or built-in filters | VRL, a real language | OTTL | Alloy expressions plus OTTL |
| Disk buffering | Yes | Yes, end-to-end acknowledgement | Via `file_storage` | Via `file_storage` |
| Metrics support | Yes, growing | Yes | Yes, native | Yes, first class |
| Traces support | Basic OTLP | Basic OTLP | Native | Native |
| Kubernetes ubiquity | Very high, the default in most distributions | Growing | High | Grafana shops |

**Fluent Bit’s case is footprint.** Three megabytes of RSS per node matters when the agent is a DaemonSet on 500 nodes, and it matters even more on edge devices and embedded systems, where it is often the only option that fits. It is the default log agent in most managed Kubernetes offerings, so it is frequently already running whether or not anyone chose it.

**Vector’s case is the transformation language.** VRL is a real, typed, fast expression language with a test harness, and log pipelines are mostly transformation work. When the job involves parsing inconsistent formats, enriching from lookup tables, computing derived fields, or routing on content, Vector is doing something the others cannot do as cleanly.

------------------------------------------------------------------------

## Fluent Bit

The pipeline is inputs, parsers, filters, outputs, wired by tag matching rather than by explicit graph edges.

``` ini
[SERVICE]
    Flush             1
    Daemon            off
    Log_Level         info
    Parsers_File      parsers.conf
    HTTP_Server       On
    HTTP_Listen       0.0.0.0
    HTTP_Port         2020
    storage.path      /var/log/flb-storage/
    storage.sync      normal
    storage.backlog.mem_limit 64M

[INPUT]
    Name              tail
    Tag               kube.*
    Path              /var/log/containers/*.log
    multiline.parser  docker, cri
    Mem_Buf_Limit     16MB
    Skip_Long_Lines   On
    DB                /var/log/flb_kube.db
    Refresh_Interval  10

[FILTER]
    Name                kubernetes
    Match               kube.*
    Kube_Tag_Prefix     kube.var.log.containers.
    Merge_Log           On
    Keep_Log            Off
    Labels              On
    Annotations         Off

[FILTER]
    Name    modify
    Match   kube.*
    Remove  kubernetes_annotations
    Rename  log message

[OUTPUT]
    Name                   loki
    Match                  kube.*
    Host                   loki
    Port                   3100
    labels                 job=fluentbit, namespace=$kubernetes['namespace_name'], app=$kubernetes['labels']['app']
    remove_keys            kubernetes, stream
    auto_kubernetes_labels off
```

**Tags and `Match` are the routing mechanism.** Every record carries a tag set by its input, and each filter and output declares which tags it applies to with a glob. There is no explicit graph; the wiring is implicit in the tag patterns, which is compact and gets confusing in a large config.

**`DB` on the tail input is the position database**, the equivalent of Promtail’s `positions.yaml`. Without it, a restart re-reads every file from the beginning and duplicates everything.

**`Mem_Buf_Limit` versus `storage.type filesystem`.** By default a backed-up output causes Fluent Bit to pause the input once the memory limit is hit, which means log loss if the file rotates meanwhile. Setting `storage.type filesystem` on the input spills to disk instead, and on any node where losing logs matters, it should be set.

**The `kubernetes` filter is the reason it is everywhere.** It takes the container log path, extracts pod and namespace, calls the API server, and attaches pod labels and annotations. Every Kubernetes logging setup needs this, and Fluent Bit’s implementation is the most battle-tested.

**Multiline is the recurring pain.** A Java stack trace is dozens of lines that belong to one event, and getting `multiline.parser` right is the difference between one useful log entry and forty useless ones. Built-in parsers cover `docker`, `cri`, `go`, `java`, `python` and `ruby`; anything else needs a custom rule set, and it is fiddly.

------------------------------------------------------------------------

## Vector

Sources, transforms and sinks, wired explicitly by naming inputs.

``` toml
[sources.docker]
type = "docker_logs"
exclude_containers = ["vector"]

[sources.host_metrics]
type = "host_metrics"
scrape_interval_secs = 15

[transforms.parse]
type = "remap"
inputs = ["docker"]
source = '''
  # Try JSON, fall back to leaving the line alone
  parsed, err = parse_json(.message)
  if err == null {
    . = merge(., object!(parsed))
  }

  # Normalise level, whatever the app called it
  .level = downcase(to_string(.level) ?? to_string(.severity) ?? "info")

  # Collapse the path so it can be a label safely
  .route = replace(to_string(.path) ?? "", r'/\d+', "/{id}")

  # Redact before it leaves the host
  if exists(.user.email) {
    .user.email = sha2(to_string!(.user.email), variant: "SHA-256")
  }
  del(.authorization)

  # Real timestamp, not ingest time
  .timestamp = parse_timestamp(.time, "%+") ?? now()
'''

[transforms.drop_noise]
type = "filter"
inputs = ["parse"]
condition = '''
  !match(to_string(.route) ?? "", r'^/(healthz|metrics|favicon.ico)$')
'''

[transforms.route_by_level]
type = "route"
inputs = ["drop_noise"]
route.errors = '.level == "error" || .level == "fatal"'

[sinks.loki]
type = "loki"
inputs = ["drop_noise"]
endpoint = "http://loki:3100"
encoding.codec = "json"
labels.job = "vector"
labels.container = "{{ container_name }}"
labels.level = "{{ level }}"
out_of_order_action = "accept"

[sinks.errors_to_s3]
type = "aws_s3"
inputs = ["route_by_level.errors"]
bucket = "log-archive"
compression = "gzip"

[sinks.prometheus]
type = "prometheus_remote_write"
inputs = ["host_metrics"]
endpoint = "http://prometheus:9090/api/v1/write"
```

**VRL is the differentiator.** It is typed, it fails at compile time rather than at 3 a.m., the `??` operator handles the pervasive “this field might not exist” problem cleanly, and there is a real test harness:

``` bash
vector vrl --input sample.json 'parse_json!(.message).level'
vector test    # runs [[tests]] blocks defined in the config
vector top     # a live TUI of throughput per component
```

Being able to unit-test a log parsing rule is not something the other agents offer, and on a pipeline that parses several inconsistent formats it changes the maintenance story completely.

**End-to-end acknowledgement** means a source does not acknowledge a record until a sink has confirmed it, which with disk buffering gives genuine at-least-once delivery. Fluent Bit’s buffering is good; Vector’s guarantee is stronger.

**Vector also does metrics natively**, including `host_metrics` as a node_exporter replacement and a `prometheus_scrape` source, so a Vector deployment can cover both signals without a second agent.

------------------------------------------------------------------------

## Problems Every Log Pipeline Has

These are agent-independent and cause most real incidents.

**Multiline.** Stack traces. Covered above, and equally painful everywhere.

**Timestamps.** Parsing the timestamp *from the line* rather than using ingest time matters the moment there is any backlog, because a replay after an outage otherwise stamps everything with the replay time and the logs land in the wrong place. Getting timezone handling right on logs that omit the offset is the follow-on problem.

**Log rotation races.** A file rotated while the agent is behind can be truncated or moved before its tail is read. Inode tracking helps, `copytruncate` in logrotate is the worst case, and the general mitigation is to read from the container runtime rather than from rotated files where possible.

**Backpressure.** When the destination is slow, an agent must drop, buffer to memory, or buffer to disk. All three choices are wrong in some situation, and the failure to actually decide is what turns a Loki outage into an OOM-killed node.

**Cardinality at the label boundary.** Every agent lets you promote a parsed field into a Loki label. Promoting the wrong one is the stream explosion from [the Loki page](../Observability/loki.llms.md), and it is the single most common way a working Loki is broken by a collection config change.

**Cost is driven by a few offenders.** Debug logging left on in one busy service routinely outweighs everything else combined. Dropping health checks and debug lines at the agent, before they cross the network, is the highest-leverage change available.

------------------------------------------------------------------------

## Choosing

**Already on Kubernetes with Fluent Bit running**: leave it. It works, it is tiny, and replacing a working log pipeline has no payoff on its own.

**Complex parsing, enrichment or routing**: Vector. VRL plus testing is a real advantage, and it grows with the pipeline.

**Grafana stack, mixed signals, one agent**: Alloy, because metrics, logs, traces and profiles are all first class.

**Vendor neutrality as a requirement**: upstream OTel Collector.

**Edge, embedded, or hundreds of nodes where footprint dominates**: Fluent Bit.

A reasonable combination is Fluent Bit as the per-node tail and Vector as an aggregation tier doing the heavy transformation, which keeps the footprint small where it is multiplied and puts the expensive work where it can be scaled independently.

------------------------------------------------------------------------

## Where Next

- [Loki](../Observability/loki.llms.md) for what these usually feed, and the label rules they must respect.
- [Alloy](../Observability/alloy.llms.md) and [the OTel Collector](../Observability/otel_collector.llms.md) for the other two options.

------------------------------------------------------------------------

Back to top
