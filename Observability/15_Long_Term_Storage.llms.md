# Long-Term Storage

Prometheus fills the disk, now what: remote write, Mimir, Thanos and VictoriaMetrics compared, downsampling, and how to choose without a migration later.

Author

Benedict Thekkel

## The Problem

A single Prometheus keeps everything on local disk with no replication and no horizontal scaling. Three limits arrive in roughly this order:

**Retention.** Fifteen days is the usual default. Capacity planning, quarterly reviews and “was this slow last Christmas too” all need months or years, and raising `retention.time` works until the disk or the compactor gives out.

**Durability.** The disk is the only copy. Losing the node loses the history, and the WAL replay after a crash on a large head block takes minutes during which nothing is answered.

**Scale.** One server ingests what one server can ingest. Past a few million active series, memory becomes the binding constraint and there is no sharding story inside Prometheus itself.

A global view across clusters is a fourth motivation and usually the one that actually forces the decision, because federating a dozen Prometheus servers by hand stops working quickly.

------------------------------------------------------------------------

## Remote Write

The common interface underneath every option here. Prometheus streams samples to a remote endpoint as they are ingested, in addition to writing locally.

``` yaml
remote_write:
  - url: http://mimir:9009/api/v1/push
    headers:
      X-Scope-OrgID: homelab            # tenant, for multi-tenant backends
    queue_config:
      capacity: 10000                   # per shard
      max_shards: 50
      min_shards: 1
      max_samples_per_send: 2000
      batch_send_deadline: 5s
      min_backoff: 30ms
      max_backoff: 5s
    write_relabel_configs:
      - source_labels: [__name__]       # ship less than you store locally
        regex: 'go_gc_.*|go_memstats_.*'
        action: drop
```

**`write_relabel_configs` is the cost control.** Remote storage is usually billed or sized by ingested samples, and a lot of what a Prometheus scrapes is never queried past a week. Dropping Go runtime internals and debug metrics at the write boundary is the easiest saving available.

**The queue shards dynamically.** Prometheus adds shards when the remote endpoint falls behind and removes them when it catches up. The metrics to watch are `prometheus_remote_storage_samples_pending`, `prometheus_remote_storage_shards`, and `prometheus_remote_storage_samples_failed_total`. A queue that is permanently at `max_shards` means the backend cannot keep up, and raising `max_shards` further usually makes it worse.

**Remote write costs local memory and CPU**, roughly 10 to 25 percent overhead. It is not free, and enabling it on a Prometheus that is already at its memory limit will push it over.

**Remote Write 2.0** adds native histogram support, metadata and exemplars in the same stream, and is worth preferring where both ends support it.

------------------------------------------------------------------------

## The Three Options

|  | Mimir | Thanos | VictoriaMetrics |
|----|----|----|----|
| Model | Push, via remote write | Pull: sidecar ships blocks to object storage | Push, via remote write |
| Lineage | Grafana, from Cortex | CNCF, independent | Independent, Go |
| Storage | Object storage | Object storage | Its own format, local disk or object storage |
| Components | Distributor, ingester, querier, store gateway, compactor, ruler | Sidecar, store, query, compact, ruler, receive | `vmsingle`, or `vminsert`/`vmstorage`/`vmselect` |
| Deduplication | On ingest, via replication | At query time, across replicas | On ingest |
| Downsampling | Yes | Yes, 5m and 1h | No, relies on compression |
| Multi-tenancy | Strong, native | Weak, by convention | Yes, in the cluster version |
| Ops complexity | High | Medium to high | Low |
| Resource use | High | Medium | Low, often dramatically |
| Query language | PromQL | PromQL | MetricsQL, a superset |

### Mimir

The Grafana answer, a fork and continuation of Cortex. Prometheus servers remote-write into it, and it handles sharding, replication, compaction and downsampling internally. It scales to billions of active series, has real multi-tenancy with per-tenant limits, and includes a ruler so recording and alerting rules can run centrally against the global view.

The cost is operational. It is a genuine distributed system with six or so component types, a hash ring, and object storage underneath. `mimir -target=all` runs it as a single binary for small deployments, which is entirely viable and is what the [LGTM stack](../Observability/lgtm_stack.llms.md) page uses, but the moment it is scaled out it is a system somebody has to own.

Choose it when the stack is already Grafana, when multi-tenancy is required, or when scale genuinely demands it.

### Thanos

The architecturally different one. Rather than receiving a push, a **sidecar** runs next to each Prometheus, uploads its completed two-hour TSDB blocks to object storage, and serves recent data from the local Prometheus. A **Querier** fans out to all sidecars and store gateways and merges the results, deduplicating across replica pairs at query time.

The consequences are real advantages. Prometheus servers stay ordinary, keeping their own data and continuing to work if Thanos is entirely down. The global view is a fan-out rather than a central ingest bottleneck. Adding a cluster means adding a sidecar.

The disadvantages are equally real. Query latency depends on the slowest sidecar. The two-hour block delay means recent data is only available from Prometheus itself, so a Prometheus that dies loses up to two hours. And deduplicating at query time is more expensive than deduplicating at ingest.

Thanos also has a `receive` component that accepts remote write, which makes it look like Mimir, and is the right choice for pushing from networks where a Querier cannot reach back in.

Choose it when Prometheus servers already exist and should stay authoritative, when the topology is many clusters with a central view, or when the operational preference is for a component you can remove without losing anything.

### VictoriaMetrics

The pragmatic one. A single binary, `vmsingle`, accepts remote write and serves PromQL, and routinely uses a fraction of the memory and disk of the alternatives for the same workload. The cluster version splits into `vminsert`, `vmstorage` and `vmselect` for horizontal scaling, and `vmagent` replaces Prometheus for scraping entirely if you want it to.

MetricsQL is a PromQL superset. Most PromQL runs unchanged, and it adds functions that are genuinely useful, but it is a compatibility claim rather than an identity, and a few edge cases differ.

It does not downsample, arguing that its compression makes it unnecessary. In practice that holds well for long retention on moderate cardinality.

Choose it when operational simplicity and resource efficiency matter most, which for a home lab or a small team is nearly always, or when a Mimir or Thanos deployment is proving expensive to run.

------------------------------------------------------------------------

## Downsampling

Raw 15-second samples are useful for a week and mostly noise after a year. Downsampling precomputes aggregates at coarser resolutions, so a year-long query reads 1-hour points instead of two million raw ones.

Thanos produces 5-minute and 1-hour resolutions and picks a level based on the query range. Mimir does the same. VictoriaMetrics does not, relying on compression instead.

**Downsampling is lossy in a way that matters.** A one-second spike disappears from a 1-hour aggregate. This is fine for capacity trends and misleading for incident forensics, so keep raw data for the period during which anyone might investigate an incident, typically two to four weeks, and downsample beyond it.

------------------------------------------------------------------------

## Federation Is Not This

Prometheus federation, where one server scrapes `/federate` on others, is sometimes suggested as a long-term-storage answer. It is not one.

``` yaml
  - job_name: federate
    honor_labels: true
    metrics_path: /federate
    params:
      'match[]':
        - '{__name__=~"job:.*"}'      # recording rules ONLY
    static_configs:
      - targets: ['prom-a:9090', 'prom-b:9090']
```

Federation is for pulling a **small, aggregated** set of series into a higher-level Prometheus, which is why the matcher above selects only recording-rule output. Pointing it at `{__name__=~".+"}` to copy everything produces a scrape that takes minutes, times out, and silently returns partial data. Use it for cross-cluster aggregates, and use remote write or Thanos for history.

------------------------------------------------------------------------

## A Practical Progression

**Small, one Prometheus.** 15 to 30 days local retention on a sized disk. Add nothing. Most setups never need to leave here, and the cost of the alternatives is real.

**Retention is the only problem.** Remote write to a `vmsingle`. One container, one volume, and Grafana gets a second datasource for long-range queries. This is the highest value per unit of effort available.

**Several Prometheus servers, need a global view.** Thanos if they should stay authoritative, Mimir if a central push model is acceptable. Both are a real operational commitment.

**Genuine scale or multi-tenancy.** Mimir, or a hosted service.

**The thing to avoid** is adopting a distributed metrics system because it is what large organisations run. Every one of these adds object storage, a component topology and a failure mode to a stack whose main job is telling you when other things break. On knowledge-lab specifically, with 20 GB of RAM shared with JupyterLab and model loading, a `vmsingle` alongside Prometheus is proportionate and a Mimir cluster is not.

------------------------------------------------------------------------

## Where Next

- [Prometheus](../Observability/prometheus.llms.md) for local retention and TSDB sizing.
- [The LGTM stack](../Observability/lgtm_stack.llms.md) for Mimir running in a compose file.
- [The landscape](../Observability/observability_landscape.llms.md) for the hosted alternatives.

------------------------------------------------------------------------

Back to top
