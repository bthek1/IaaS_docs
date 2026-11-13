# SLOs and Alerting Practice

RED, USE, the four golden signals, error budgets and multi-window burn-rate alerts: the discipline that decides what to measure and what is allowed to wake someone.

Author

Benedict Thekkel

## The Problem This Solves

A stack that collects everything gives you thousands of numbers and no opinion about which matter. The predictable result is a dashboard wall nobody reads and an alert channel everyone mutes, and the muting is rational: if most notifications do not require action, ignoring them is the correct response.

The frameworks below are all answers to the same two questions. **What should I measure?** and **when is a human allowed to be interrupted?** They are cheap to adopt and they are what separates a monitoring setup from an on-call practice.

------------------------------------------------------------------------

## The Four Golden Signals

From Google’s SRE book, and the most durable of the three framings.

| Signal | Means | Typical query |
|----|----|----|
| **Latency** | How long requests take. Split successful from failed | `histogram_quantile(0.99, sum by (le) (rate(duration_bucket[5m])))` |
| **Traffic** | Demand on the system | `sum(rate(http_requests_total[5m]))` |
| **Errors** | Rate of failing requests, explicit and implicit | `sum(rate(http_requests_total{status=~"5.."}[5m]))` |
| **Saturation** | How full the constrained resource is | `1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m]))` |

**Splitting failed from successful latency matters more than it sounds.** A service that fails fast has excellent latency numbers and is completely broken. Averaging the two together produces a graph that improves as the system degrades, which is worse than no graph.

**Saturation needs the right resource.** It is whichever thing runs out first, and it is frequently not CPU: a connection pool, a thread pool, a queue, disk IOPS, memory bandwidth. Measuring CPU because it is easy to measure is how a service that is entirely blocked on a 20-connection database pool looks idle.

------------------------------------------------------------------------

## RED And USE

Two specialisations, for two kinds of thing.

**RED, for anything serving requests.** Rate, Errors, Duration. It is the golden signals minus saturation, and it applies uniformly to every service, which is what makes it the basis for a generic service dashboard. Every service gets the same three panels and anyone can read any service’s dashboard without learning it.

**USE, for every resource.** Utilisation, Saturation, Errors. Utilisation is the fraction of time busy, saturation is the queue of work waiting, errors are the count of failed operations. Applied to CPU, memory, disk, network, and any pool in the application.

The distinction is the useful part: **RED describes what users experience, USE describes why.** RED alerts, USE explains. A page on high disk utilisation is a cause-based alert that may harm nobody; a page on the error ratio users see is a symptom, and the disk graph is what you open next.

``` promql
# RED, as three queries, per service
sum by (service) (rate(http_requests_total[5m]))
sum by (service) (rate(http_requests_total{status=~"5.."}[5m])) / sum by (service) (rate(http_requests_total[5m]))
histogram_quantile(0.99, sum by (le, service) (rate(http_request_duration_seconds_bucket[5m])))

# USE, for a connection pool
db_pool_connections_active / db_pool_connections_max          # utilisation
db_pool_wait_queue_depth                                      # saturation
rate(db_pool_timeouts_total[5m])                              # errors
```

------------------------------------------------------------------------

## SLI, SLO, SLA

**An SLI is a measurement** of something users care about, expressed as a ratio of good events to valid events.

    availability SLI = successful requests / all requests
    latency SLI      = requests faster than 300ms / all requests
    freshness SLI    = pipeline runs completed within 1h / all runs

**An SLO is a target for an SLI over a window.** “99.9 percent of requests succeed, measured over 30 days.”

**An SLA is a contract** with consequences attached, and is always looser than the internal SLO, because the internal target needs headroom before anyone owes anybody money.

### Choosing An SLI

The test is whether a change in the number reflects a change in user happiness. Requests-per-second fails that test; a 50 percent traffic drop might be a Sunday. Error ratio passes it.

Prefer **ratios of events** over averages and gauges. A ratio is naturally bounded, aggregates correctly, and makes the error-budget arithmetic below work. CPU utilisation is not an SLI.

**Measure as close to the user as possible.** A server-side error ratio misses the requests that never reached the server, which is exactly what a load balancer outage looks like. Client-side or load-balancer-side measurement catches more, at the cost of being harder to collect.

------------------------------------------------------------------------

## Error Budgets

The reframing that makes SLOs useful rather than aspirational. A 99.9 percent target over 30 days permits 0.1 percent failure, and that allowance is a **budget** to be spent.

    30 days              = 43,200 minutes
    99.9% availability   = 43.2 minutes of failure allowed per 30 days
    99.95%               = 21.6 minutes
    99.99%               = 4.32 minutes

| SLO     | Downtime per 30 days | Per year     |
|---------|----------------------|--------------|
| 99%     | 7.2 hours            | 3.65 days    |
| 99.5%   | 3.6 hours            | 1.83 days    |
| 99.9%   | 43.2 minutes         | 8.77 hours   |
| 99.95%  | 21.6 minutes         | 4.38 hours   |
| 99.99%  | 4.32 minutes         | 52.6 minutes |
| 99.999% | 26 seconds           | 5.26 minutes |

Two consequences follow, and both are cultural rather than technical.

**Budget remaining is permission to take risk.** Plenty of budget left means ship the risky change. Budget exhausted means stop shipping features and spend the effort on reliability. This converts an argument between product and operations into a number both sides agreed on in advance.

**100 percent is the wrong target.** Each additional nine costs roughly an order of magnitude more, and no user can tell 99.99 from 99.999 through their own network. Picking a target above what users can perceive spends engineering effort on nothing.

**Pick the target from observed reality, not from aspiration.** Measure the current SLI for a month first. A service currently at 99.5 percent with a 99.99 percent target produces an alert that is always firing, which is the same as no alert.

------------------------------------------------------------------------

## Burn Rate Alerting

The mechanism that replaces arbitrary thresholds. **Burn rate** is how fast the error budget is being consumed relative to the rate that would exhaust it exactly at the end of the window.

    burn rate = observed error ratio / (1 - SLO)

    At a 99.9% SLO (budget 0.1%):
      0.1% errors  -> burn rate 1    -> exhausts the budget exactly at 30 days
      1%   errors  -> burn rate 10   -> exhausts it in 3 days
      10%  errors  -> burn rate 100  -> exhausts it in about 7 hours

Alerting on burn rate rather than on a fixed error percentage means the severity is automatically proportional to the consequence. A slow trickle and a total outage are the same alert with different urgency, instead of two thresholds somebody guessed.

### The Multi-Window Pattern

A single window forces the usual bad trade: short windows are fast and noisy, long windows are quiet and slow. The multi-window, multi-burn-rate pattern from the SRE workbook uses both at once. **The long window decides whether it is real; the short window decides whether it is still happening.**

``` yaml
groups:
  - name: slo-api-availability
    rules:
      # Precompute the error ratio at several windows
      - record: job:slo_errors:ratio5m
        expr: |
          sum(rate(http_requests_total{job="api",status=~"5.."}[5m]))
            / sum(rate(http_requests_total{job="api"}[5m]))
      - record: job:slo_errors:ratio30m
        expr: |
          sum(rate(http_requests_total{job="api",status=~"5.."}[30m]))
            / sum(rate(http_requests_total{job="api"}[30m]))
      - record: job:slo_errors:ratio1h
        expr: |
          sum(rate(http_requests_total{job="api",status=~"5.."}[1h]))
            / sum(rate(http_requests_total{job="api"}[1h]))
      - record: job:slo_errors:ratio6h
        expr: |
          sum(rate(http_requests_total{job="api",status=~"5.."}[6h]))
            / sum(rate(http_requests_total{job="api"}[6h]))
      - record: job:slo_errors:ratio3d
        expr: |
          sum(rate(http_requests_total{job="api",status=~"5.."}[3d]))
            / sum(rate(http_requests_total{job="api"}[3d]))

      # Fast burn: 14.4x burn rate consumes 2% of a 30d budget in 1 hour. Page.
      - alert: SLOErrorBudgetFastBurn
        expr: |
          job:slo_errors:ratio1h > (14.4 * 0.001)
            and
          job:slo_errors:ratio5m > (14.4 * 0.001)
        for: 2m
        labels: {severity: page}
        annotations:
          summary: "API burning error budget 14x too fast"

      # Slow burn: 6x consumes 5% in 6 hours. A ticket, not a page.
      - alert: SLOErrorBudgetSlowBurn
        expr: |
          job:slo_errors:ratio6h > (6 * 0.001)
            and
          job:slo_errors:ratio30m > (6 * 0.001)
        for: 15m
        labels: {severity: ticket}
        annotations:
          summary: "API will exhaust its error budget in under a week"
```

The standard four-tier configuration for a 30-day window:

| Burn rate | Long window | Short window | Budget consumed | Action |
|-----------|-------------|--------------|-----------------|--------|
| 14.4      | 1 hour      | 5 min        | 2%              | Page   |
| 6         | 6 hours     | 30 min       | 5%              | Page   |
| 3         | 1 day       | 2 hours      | 10%             | Ticket |
| 1         | 3 days      | 6 hours      | 10%             | Ticket |

**The short window is the reset mechanism.** Without it, a 1-hour window keeps an alert firing for an hour after the incident ended, because the window still contains the errors. Requiring the short window to also be over threshold means the alert clears within minutes of recovery, which is the behaviour that makes people trust it.

This is why `job:slo_errors:ratio*` are recording rules. Evaluating a `[3d]` range on every rule evaluation is expensive, and the rules above reference the precomputed series.

------------------------------------------------------------------------

## What Deserves A Page

Three tests, all of which must pass.

**Urgent.** If it can wait until morning, it is a ticket. Route it as one rather than paging and expecting good judgement at 3 a.m.

**Actionable.** There is something the recipient can do now. An alert whose runbook says “keep an eye on it” teaches people that alerts can be ignored, and that lesson generalises.

**Symptom, not cause.** Alert on what users experience. High CPU that harms nobody is not an incident, and the cause-based alert misses every outage with a different cause.

The corollary is that **most existing alerts should be deleted.** A useful audit is to list every alert that fired in the last quarter and ask, for each, what the recipient did. Alerts where the answer is “nothing” or “acknowledged it” are noise and are actively harmful, because they are the reason the real one gets missed.

### Writing The Annotation

The person reading it is half asleep and does not have context. The summary should name what is broken, where, and how badly, in one line. The description should say what to check first. A runbook link should point at something that exists.

``` yaml
annotations:
  summary: "api in prod: 12% of requests failing (budget burn 120x)"
  description: >
    Started 14:02 UTC. Check the last deploy (argocd app api), then the
    postgres connection pool saturation panel, then upstream auth-service.
  runbook_url: https://github.com/bthek1/Knowledge/issues/19
  dashboard_url: https://grafana/d/api-overview
```

------------------------------------------------------------------------

## Applying This To A Home Lab

Most of the above is written for a team with an on-call rotation, and scales down with some honesty about what changes.

**There is no rotation, so “page” means a phone notification you will genuinely act on.** Keep that list to real problems: the host is down, the disk will fill, the backup has not succeeded in 48 hours, the certificate expires in a week. Everything else is a dashboard.

**SLOs are still useful, as a measurement rather than a commitment.** Knowing that JupyterLab was reachable 99.2 percent of last month is genuinely informative, and burn-rate alerting is overkill when the budget is nobody’s to spend.

**Saturation is the signal that matters most here**, because the constraint is real and known: 4 vCPU, 20 GB RAM, 12 GB VRAM on knowledge-lab. A memory alert at 85 percent of container RAM is worth more than any latency percentile, since the actual failure mode is a notebook OOMing the container.

``` promql
# The alerts that earn their place on a small box
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes < 0.15
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[6h], 24*3600) < 0
up == 0
time() - node_boot_time_seconds < 300            # it rebooted and you did not ask it to
DCGM_FI_DEV_FB_USED / DCGM_FI_DEV_FB_TOTAL > 0.95   # VRAM nearly full
```

------------------------------------------------------------------------

## Where Next

- [Alerting](../Observability/alerting.llms.md) for the rule and routing mechanics.
- [PromQL](../Observability/promql.llms.md) for the recording rules these depend on.
- [Exporters](../Observability/exporters_and_instrumentation.llms.md) for what to instrument to make RED and USE possible.

------------------------------------------------------------------------

Back to top
