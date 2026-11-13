# Observability Overview

What the signals are, how they differ from monitoring, and how the pieces of a telemetry stack fit together.

Author

Benedict Thekkel

## Monitoring Answers Known Questions, Observability Answers New Ones

Monitoring is the practice of watching a fixed set of things you already decided matter: CPU above 90 percent, disk above 80 percent, the health check returning non-200. It works when failures repeat, and it is why a server-era ops team could get by with Nagios and a pager.

Observability is the property of a system that lets you ask a question nobody anticipated, without shipping new code to answer it. The canonical framing is that monitoring handles known unknowns and observability handles unknown unknowns. In practice the distinction shows up the first time somebody asks “why are checkout requests slow, but only for users on the Android app, in Sydney, who have more than four items in the cart”. No dashboard was built for that question. Either the telemetry carries enough dimensions to slice it after the fact, or the investigation stalls and turns into guesswork.

That is the whole design pressure behind this section. Every tool here is a bet about which dimensions to keep, at what cost.

------------------------------------------------------------------------

## The Signals

Three signals are conventional, and a fourth is now standard enough to count.

| Signal | Shape | Answers | Cost driver | Store |
|----|----|----|----|----|
| Metrics | Numbers over time, tagged with labels | Is it broken, how broken, for how long | Unique label combinations | Prometheus, Mimir, VictoriaMetrics |
| Logs | Timestamped events, structured or not | What exactly happened in this one case | Bytes ingested and retained | Loki, Elasticsearch, ClickHouse |
| Traces | A causal tree of spans across services | Where the time went, which hop failed | Spans per second, sampling rate | Tempo, Jaeger |
| Profiles | Stack samples over time | Which line of code burns the CPU or the heap | Sample rate times process count | Pyroscope, Parca |

The signals are not interchangeable, and the usual failure is trying to make one do another’s job.

**Metrics are cheap and lossy.** A counter costs a few bytes per scrape no matter how many requests it counts, which is why metrics are what you alert on. The loss is that a metric has already thrown away the individual events. Once `http_requests_total{status="500"}` ticks up, there is no way to get back to which request failed.

**Logs are expensive and exact.** They keep the individual event, so they answer the specific question, and the bill scales with how much you write. A single verbose service can cost more to store than the rest of the fleet combined.

**Traces connect the two across process boundaries.** Metrics tell you the checkout endpoint is slow, logs tell you what one slow request did, and a trace tells you that 900 ms of the 1.1 s was spent in a downstream auth call that nobody suspected. Traces are the only signal that survives the jump from a monolith to a distributed system intact.

**Profiles go one level below traces**, from “which service” to “which function”. They are the answer to a CPU graph that is high with no obvious culprit.

------------------------------------------------------------------------

## Cardinality Is The Constraint That Shapes Everything

A metric is not one series. It is one series per unique combination of label values.

    http_requests_total{method="GET", route="/api/users", status="200"}
    http_requests_total{method="GET", route="/api/users", status="500"}
    http_requests_total{method="POST", route="/api/users", status="201"}

Three label pairs, three series. Add a `user_id` label to a service with 100,000 users and you have just asked the database to hold up to 100,000 times more series, each with its own index entry, its own memory footprint in the head block, and its own compaction cost. This is the single most common way to take down a Prometheus, and it usually arrives as a well-meant pull request adding a label that seemed useful.

The rule of thumb: a label is safe when its value set is small, bounded, and known in advance. `status`, `method`, `region`, `instance` are fine. `user_id`, `request_id`, `session_id`, `url_with_query_string`, `error_message` are not. Anything unbounded belongs in a log line or a trace span attribute, where the cost model is bytes rather than series.

This constraint is exactly what Loki was designed around. Loki indexes only labels and leaves the log body unindexed, which makes it cheap to store and means a high-cardinality field like a request ID goes in the message body, not the label set. Honeycomb takes the opposite bet, building a columnar store where high cardinality is the point. Both are coherent answers to the same pressure.

------------------------------------------------------------------------

## Pull Versus Push

Prometheus pulls. It is configured with a list of targets, and on an interval it makes an HTTP request to each one and reads a text exposition of the current values. Most of the rest of the world pushes: the application or an agent sends telemetry to a collector.

|  | Pull (Prometheus) | Push (OTLP, StatsD, most vendors) |
|----|----|----|
| Who initiates | The server, on a schedule | The client, when it has data |
| Target discovery | Service discovery is mandatory and central | Clients need an endpoint and credentials |
| Health signal | Free: a target that fails to scrape is `up == 0` | Needs a separate heartbeat |
| Short-lived jobs | Awkward, needs a Pushgateway | Natural |
| Firewalls | Server must reach every target | Clients must reach one endpoint |
| Backpressure | Server controls the rate | Client can flood the server |

Neither is correct in general. Pull suits a fleet of long-lived services inside one network boundary, which is why it fits Kubernetes so neatly: the API server already knows every pod, so service discovery is solved. Push suits batch jobs, serverless, mobile clients, and anything outside the network perimeter.

In practice most stacks run both. An OTel Collector accepts pushed OTLP from applications and simultaneously scrapes Prometheus endpoints, then writes everything onward through the same pipeline.

------------------------------------------------------------------------

## How The Pieces Fit

A telemetry stack has four layers, and almost every tool in this section belongs to exactly one of them.

      instrumentation   application code emits: SDK, client library, or automatic
            |
      collection        an agent on the node or a gateway: Alloy, OTel Collector,
            |           Fluent Bit, Vector. Batches, transforms, routes, retries.
            |
      storage           one backend per signal, or one that takes several:
            |           Prometheus / Mimir, Loki, Tempo, Pyroscope
            |
      query and alert   Grafana for reading, Alertmanager for routing

Read the layers rather than the product names and the field stops looking crowded. Alloy and the OTel Collector are the same layer, and Alloy is literally a distribution of the Collector. Fluent Bit and Vector are that layer too, approached from the logging side. Mimir, Thanos and VictoriaMetrics are all the storage layer answering the same question about Prometheus retention. Datadog and SigNoz are all four layers sold as one product.

The value of splitting the layers is that they can be replaced independently. Instrumenting with OpenTelemetry rather than a vendor SDK means the storage layer is swappable later, which is the entire argument for OTel and the subject of its page.

------------------------------------------------------------------------

## Where To Start Reading

The pages in this folder are ordered so each depends only on the ones above it.

1.  **Metrics first** (Prometheus, exporters, PromQL, alerting). Metrics are what you alert on, so this is the half that has to work before anything else matters.
2.  **Then the other signals** (Loki, LogQL, Tempo, Pyroscope), which share the concepts but differ in cost model.
3.  **Then collection** (OpenTelemetry, the Collector, Alloy, Fluent Bit and Vector), which is where the signals converge onto one pipeline.
4.  **Then Grafana, provisioning and scale**, the read side and the operational answers.
5.  **Finally practice and landscape**, which is what turns a pile of tools into something you can be on call for.

If you want a running stack on screen before any of that, the LGTM Stack page at the end has the whole thing in a single compose file.

------------------------------------------------------------------------

Back to top
