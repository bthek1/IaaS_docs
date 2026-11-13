# Loki

Log aggregation built on the bet that indexing only labels is enough: the storage model, why it is not Elasticsearch, and how to run it without drowning it.

Author

Benedict Thekkel

## The Bet

Elasticsearch builds a full-text inverted index over every log line. That makes arbitrary search fast and makes ingestion expensive: the index is often larger than the data, and the cluster that holds it is the most operationally demanding thing in most stacks.

Loki indexes **only the labels**. The log body is compressed and stored in chunks, never indexed. A query therefore works in two stages: use the label index to find the handful of streams that could contain the answer, then brute-force scan those chunks.

The bet is that the label set narrows the search enough that scanning is cheap. It pays off when queries are scoped by service, namespace and time, which is how people actually search logs during an incident. It fails badly when someone asks a question that cannot be narrowed, such as searching every log in the cluster for one request ID across a week.

The payoff is cost. Loki’s index is tiny, chunks go to object storage, and the whole thing runs on S3 plus some stateless processes. Running it is closer to running a web service than to running a database.

------------------------------------------------------------------------

## Streams, Labels And Chunks

A **stream** is a unique combination of labels. Each stream has its own chunk sequence, and log lines within a stream must arrive in timestamp order.

    {job="api", namespace="prod", level="error"}     <- one stream
    {job="api", namespace="prod", level="info"}      <- a different stream

**Cardinality is the constraint again, and it bites harder than in Prometheus.** Each stream is a separate chunk being filled, held in memory in the ingester until it is flushed. Ten thousand streams means ten thousand open chunks. A label with unbounded values (a request ID, a user ID, a pod name in a cluster with high pod churn) multiplies streams until the ingesters OOM.

The working rule: **keep the label count in single digits and every label’s value set small.** Around ten to fifteen labels is already a lot, and each additional label multiplies rather than adds.

What goes in labels: `job`, `namespace`, `app`, `container`, `level`, `env`, `cluster`. What does not: request IDs, trace IDs, user IDs, status codes on a high-traffic service, and anything extracted from the message body. Those are found by filtering the body at query time, which is what Loki is built to do fast.

### Structured Metadata

Loki 3.0 added structured metadata, a per-line key-value attachment that is stored alongside the line but not indexed as a label and does not create a new stream. This is the correct home for exactly the high-cardinality fields that used to force a bad choice between a label explosion and losing the field entirely. Trace IDs are the intended case, and it is what makes clicking from a log line to its trace work without a `trace_id` label.

------------------------------------------------------------------------

## The Storage Model

      distributor   validates, applies rate limits, hashes labels, forwards to ingesters
           |
      ingester      builds chunks in memory per stream, flushes on size or age
           |
      object store  chunks plus the index (TSDB format since 2.8). S3, GCS, or local disk
           |
      querier       resolves labels through the index, downloads chunks, greps them

A single-binary Loki runs all of these in one process and is entirely appropriate up to a few hundred GB a day. The microservices mode splits them for independent scaling, and simple scalable mode is the middle ground that separates read from write. Start with single binary.

**Chunks flush on size or idle time, so recent logs live only in ingester memory.** A crashed ingester loses whatever had not flushed unless the write-ahead log is enabled. Enable the WAL.

**The chunk store is append-only and immutable.** Deleting specific log lines is not a normal operation; retention is enforced by dropping whole chunks past a cutoff. Plan for that when logs contain anything that might need to be removed on request.

------------------------------------------------------------------------

## Configuration Worth Understanding

``` yaml
auth_enabled: false              # true means every request needs X-Scope-OrgID

server:
  http_listen_port: 3100

common:
  instance_addr: 127.0.0.1
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb               # the current index type
      object_store: filesystem  # or s3, gcs, azure
      schema: v13               # the current schema
      index:
        prefix: index_
        period: 24h

limits_config:
  retention_period: 744h                  # 31 days
  ingestion_rate_mb: 10                   # per tenant, per distributor
  ingestion_burst_size_mb: 20
  max_streams_per_user: 10000
  max_label_names_per_series: 15
  reject_old_samples: true
  reject_old_samples_max_age: 168h
  allow_structured_metadata: true

compactor:
  working_directory: /loki/compactor
  retention_enabled: true                 # required, or retention_period does nothing
  delete_request_store: filesystem
```

**`schema_config` is append-only.** Changing an existing entry corrupts the ability to read old data. To migrate schema or storage, add a new entry with a future `from` date and leave the old one in place forever. This is the single most consequential thing to get right, because it cannot be undone.

**`retention_enabled: true` on the compactor is mandatory.** Setting `retention_period` alone does nothing at all; the compactor is what applies it. Discovering this after a year of accumulation is a common and expensive surprise.

**Out-of-order writes.** Loki historically rejected any line older than the newest one in its stream, which produced constant `entry out of order` errors from multi-source collectors. Recent versions accept out-of-order writes within a window, but `reject_old_samples_max_age` still bounds how far back anything can be written, so a collector catching up after a long outage will silently drop its backlog.

------------------------------------------------------------------------

## Getting Logs In

| Agent | Notes |
|----|----|
| Alloy | The current Grafana recommendation. Covered in [its own page](../Observability/alloy.llms.md) |
| Promtail | The original Loki agent. Deprecated as of Loki 3.x, feature-frozen, support ends 2026. Still everywhere |
| OTel Collector | Via the `loki` exporter, or OTLP directly, which Loki now accepts natively |
| Fluent Bit, Vector | Both have Loki outputs. See [log collectors](../Observability/log_collectors.llms.md) |
| Docker driver | `loki` logging driver on the daemon, no agent process at all |

Promtail is deprecated but still what most existing setups run, and its config is the clearest illustration of the model, so it is worth reading even when deploying Alloy.

``` yaml
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml     # how far into each file it has read

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: system
    static_configs:
      - targets: [localhost]
        labels:
          job: varlogs
          host: knowledge-lab
          __path__: /var/log/*log      # the file glob to tail

  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: [__meta_docker_container_name]
        regex: "/(.*)"
        target_label: container
      - source_labels: [__meta_docker_container_log_stream]
        target_label: stream
    pipeline_stages:
      - json:
          expressions:
            level: level
            msg: message
      - labels:
          level:                        # promote ONLY the bounded field
      - timestamp:
          source: time
          format: RFC3339Nano
```

The `scrape_configs` and `relabel_configs` structure is deliberately identical to Prometheus. `__path__` is the Promtail-specific label naming the files to tail, and `positions.yaml` is what stops a restart from re-sending every log on disk.

**`pipeline_stages` run per line at collection time** and are where parsing, timestamp extraction, filtering and label promotion happen. The `labels` stage is the dangerous one: every field promoted there becomes part of the stream identity. Promoting `level` is fine because there are five values. Promoting `status_code` or `user_id` creates the stream explosion described above.

------------------------------------------------------------------------

## Loki Versus Elasticsearch

|  | Loki | Elasticsearch / OpenSearch |
|----|----|----|
| Index | Labels only | Full text, every field |
| Ingest cost | Low | High: analysis plus index writes |
| Storage | Compressed chunks in object storage | Index segments on fast local disk |
| Arbitrary search | Brute-force scan of matching streams | Fast, that is the whole point |
| Scoped search | Fast | Fast |
| Aggregations over fields | Limited, computed at query time | Rich and fast |
| Operational burden | Low, mostly stateless | High: shards, heap, cluster state |
| Cost at 1 TB a day | Object storage plus some compute | A meaningful cluster |

Choose Loki when logs are read by service and time window, when cost matters more than search sophistication, and when Grafana is already the interface. Choose Elasticsearch when logs are searched rather than tailed, when arbitrary full-text queries across everything are the normal access pattern, or when the logs are the product rather than a debugging aid. ClickHouse-backed stores are increasingly the third answer, offering real aggregation at a cost closer to Loki’s.

------------------------------------------------------------------------

## Operating It

**Watch the stream count.** `sum(loki_ingester_memory_streams)` is the number to alert on, against `max_streams_per_user`. It climbs slowly and then a deploy adds a label and it climbs fast.

**Expect 429s and understand them.** `ingestion_rate_mb` is per tenant per distributor, and hitting it drops logs. The error text names the limit involved, which is the fastest path to the right config key.

**`too many outstanding requests`** means the query is scanning far more than it should, almost always because the label selector is too broad. Narrow the stream selector before raising any limit.

**Query performance is about how much gets scanned**, so the first filter in a query should be the most selective one. That is a [LogQL](../Observability/logql.llms.md) concern and is covered there.

------------------------------------------------------------------------

## Where Next

- [LogQL](../Observability/logql.llms.md) for querying, including metric queries over log lines.
- [Alloy](../Observability/alloy.llms.md) for the current collection agent.
- [The LGTM stack](../Observability/lgtm_stack.llms.md) for a running compose file with Loki in it.

------------------------------------------------------------------------

Back to top
