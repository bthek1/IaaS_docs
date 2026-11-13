# The Observability Landscape

Everything this section does not run: the commercial platforms, the all-in-one open source stacks, the eBPF layer, and what each one is actually the answer to.

Author

Benedict Thekkel

## How To Read This Field

The tooling looks crowded and is not. Almost every product is one of four things, and naming which one collapses most of the confusion.

1.  **A storage backend for one signal.** Prometheus, Loki, Tempo, Pyroscope, Elasticsearch, Jaeger.
2.  **A collection agent.** OTel Collector, Alloy, Fluent Bit, Vector, Telegraf.
3.  **A query and visualisation front end.** Grafana, Kibana.
4.  **All three sold together.** Datadog, New Relic, SigNoz, Dynatrace.

The fifth category, and the genuinely new one, is **eBPF-based collection**, which gets telemetry without instrumenting anything.

The other useful axis is the **cost model**, because it predicts behaviour better than any feature list. Per-host pricing punishes small services and rewards big ones. Per-GB-ingested punishes verbose logging and creates an incentive to log less, which is sometimes the wrong incentive. Self-hosted open source moves the cost to engineering time, which is real but easier to ignore on a balance sheet.

------------------------------------------------------------------------

## Commercial Platforms

| Product | Strength | Cost model | Known for |
|----|----|----|----|
| **Datadog** | Breadth. 900+ integrations, every signal, good defaults | Per host, plus per signal, plus indexed spans, plus custom metrics | The most complete product and bills that arrive as a shock |
| **New Relic** | APM depth, generous free tier | Per GB ingested, plus per user | Pivoted from a per-host model to ingest pricing |
| **Dynatrace** | Automatic instrumentation and dependency mapping | Per hour of host memory, plus units | The OneAgent doing everything with minimal setup |
| **Splunk** | Log search at scale, security convergence | Per GB ingested, or per workload | The incumbent in large enterprises, and SIEM |
| **Honeycomb** | High-cardinality event analysis | Per event | Arguing observability means arbitrary slicing, not dashboards |
| **Grafana Cloud** | The stack on this site, hosted | Per series, per GB logs and traces, per user | The managed version of everything here |
| **Chronosphere** | Cost control on metrics at scale | Per series ingested | Built around reducing cardinality before it is stored |
| **Sentry** | Error tracking with stack traces and release tracking | Per event | Developers adopting it without asking operations |

**Datadog is the default enterprise answer** and deserves to be understood even if never used. Its integration coverage is real, its onboarding is fast, and its billing has four or five dimensions that grow independently, which is why cost control is an ongoing discipline rather than a purchase decision. Custom metrics are billed per unique series, so the cardinality lesson from [the overview](../Observability/observability_overview.llms.md) arrives as an invoice.

**Honeycomb represents a genuinely different position.** Rather than pre-aggregated metrics plus separate logs and traces, it stores wide structured events and makes arbitrary high-cardinality grouping fast. “Show me p99 latency grouped by customer ID, build number and feature flag” is a normal query rather than an impossibility. If the recurring frustration is that the dashboards never answer the actual question, that is the argument it is making.

**Sentry is worth noting separately** because it solves a problem the rest of this section barely touches: an exception with the stack trace, the source line, the release, the user affected, and whether it is new or a regression. Most teams end up with Sentry plus something else, and that is a reasonable outcome rather than a duplication.

------------------------------------------------------------------------

## All-In-One Open Source

The response to LGTM’s component count and to commercial pricing. All are OTel-native, most are ClickHouse-backed, and the pitch is one deployment instead of five.

| Project | Notes |
|----|----|
| **SigNoz** | The most mature. ClickHouse, all three signals, APM views, its own UI. The clearest Datadog alternative |
| **Uptrace** | Similar, lighter, smaller scope |
| **HyperDX** | ClickHouse, strong on session replay plus logs and traces together |
| **OpenObserve** | Rust, claims very low storage cost, single binary |
| **Coroot** | eBPF collection plus automatic service maps and inferred SLOs. Closest to zero-effort |

**ClickHouse underneath is the common thread**, and it is why these exist now. A columnar store that does fast aggregation over high-cardinality data handles logs, traces and metrics acceptably in one engine, which removes the reason to run three specialised stores. That is the architectural bet, and it is a credible one.

**Choose one over LGTM** when the operational cost of five components is the dominant concern, when a single team owns everything, or when wanting SQL over telemetry matters. **Choose LGTM** when each component needs to scale independently, when Grafana is already the standard, or when the flexibility of separate, replaceable backends is worth the assembly.

------------------------------------------------------------------------

## The eBPF Layer

eBPF runs sandboxed programs in the Linux kernel, which means telemetry can be gathered from outside every process, with no code change, no SDK, no restart and no language support matrix.

| Tool | Does |
|----|----|
| **Cilium / Hubble** | Network policy and flow observability, service-to-service traffic |
| **Pixie** | Auto-collects protocol traces, metrics and profiles in Kubernetes. Query with PxL |
| **Grafana Beyla** | Zero-code OTel instrumentation: RED metrics and traces from HTTP and gRPC traffic |
| **Odigos** | Detects languages and injects OTel instrumentation automatically |
| **Parca**, **Pyroscope eBPF** | Whole-fleet continuous profiling |
| **Coroot** | Full stack built on eBPF collection |

**What it genuinely solves**: telemetry from services nobody will instrument. Legacy code, third-party binaries, a dozen languages, teams without capacity. Pointing Beyla at a process and getting RED metrics within minutes is a real capability.

**What it does not solve**: semantics. eBPF sees syscalls, sockets and stacks. It cannot know that this request was a checkout for a returning customer, or which business operation a span belongs to, because that information exists only in the application’s own head. Context propagation across services is also hard to do from the kernel, so distributed traces from eBPF are often less complete than instrumented ones.

The realistic position is that eBPF is an excellent floor and a poor ceiling. Use it to get every service to a baseline without asking anybody for anything, and instrument the services that matter properly.

------------------------------------------------------------------------

## The Rest Of The Field

**Metrics storage**: InfluxDB and Telegraf (broad plugin collection, a different query model), Graphite (legacy, still everywhere), TimescaleDB (Postgres extension, when SQL matters), M3 (Uber’s, largely superseded).

**Logs**: Graylog (full management layer over a search backend), Quickwit (cheap search on object storage), ClickHouse directly (increasingly the default answer for cost-sensitive logging at scale).

**Tracing**: Jaeger (the CNCF standard, covered in [Tempo](../Observability/tempo.llms.md)), Zipkin (older, still in Spring shops).

**SLO tooling**: Sloth and Pyrra generate the full set of burn-rate recording and alerting rules from an SLO definition, which is the tedious part of [SLOs and alerting practice](../Observability/slos_and_alerting_practice.llms.md). OpenSLO is the vendor-neutral specification. Nobl9 is the commercial platform.

**On-call and incident response**: PagerDuty (the incumbent), Opsgenie (Atlassian), Incident.io and Rootly (incident management rather than paging), Grafana OnCall.

**Synthetic and uptime**: Blackbox Exporter (in [exporters](../Observability/exporters_and_instrumentation.llms.md)), Uptime Kuma (self-hosted, excellent for a home lab), Checkly (browser-based checks as code), Grafana Synthetic Monitoring.

**Real user monitoring**: Grafana Faro, Sentry, and the OTel browser SDK. This is the only way to see what users actually experienced, including the requests that never reached a server.

**Cost**: OpenCost (CNCF standard for Kubernetes cost allocation), Kubecost, Infracost (cost diff in a pull request).

**Kubernetes-specific**: kube-state-metrics, node-exporter and cAdvisor (the standard trio), Robusta (alert enrichment), Goldilocks (right-sizing recommendations), k8sgpt (automated cluster diagnosis).

**Chaos engineering**: Litmus and Chaos Mesh, both CNCF. Worth mentioning here because a deliberately induced failure is the only honest test of whether the alerting works.

------------------------------------------------------------------------

## Where This Stack Sits

The pages in this folder describe self-hosted LGTM plus OpenTelemetry. That is one coherent position among several, and it is worth being explicit about the tradeoff.

**It is chosen when** cost matters more than convenience, when the data should stay on infrastructure you control, when the components need to be replaceable, and when learning how the pieces work is itself a goal. For a home lab it is close to free, and the knowledge transfers directly to any employer running the same stack, which is a large fraction of them.

**It is the wrong choice when** the engineering time to assemble and operate it is worth more than a licence, when the team is small and nobody wants to own it, or when a vendor’s integrations would cover in an afternoon what would otherwise take weeks.

**What makes either choice survivable is OpenTelemetry.** Instrumenting with OTel means the storage and query layers stay replaceable, so the expensive, slow, permanent part of the work is not tied to any of the products named on this page. That is the single most durable decision in the whole section.

------------------------------------------------------------------------

## Not Covered Anywhere Here

Named rather than implied, so the gaps read as deliberate:

- **Security observability and SIEM.** Falco, Wazuh, and the log-pipeline-to-detection path. Adjacent and genuinely different.
- **Database-specific deep monitoring.** `pg_stat_statements`, query plan analysis, lock analysis. The exporters give shape; the depth is a database topic.
- **Frontend performance.** Core Web Vitals, real user monitoring in practice, session replay.
- **Cost observability in practice**, beyond naming the tools.
- **Running any of these at real scale.** Every page here assumes one cluster or one machine.

------------------------------------------------------------------------

## Where Next

- [The overview](../Observability/observability_overview.llms.md) for the concepts these products all implement.
- [SLOs and alerting practice](../Observability/slos_and_alerting_practice.llms.md) for the discipline that matters regardless of which product is used.
- [The LGTM stack](../Observability/lgtm_stack.llms.md) for this section’s own stack, running.

------------------------------------------------------------------------

Back to top
