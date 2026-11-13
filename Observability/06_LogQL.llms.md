# LogQL

Loki’s query language: stream selectors, line and label filters, parsers, and the metric queries that turn log lines into graphs and alerts.

Author

Benedict Thekkel

## The Shape Of A Query

Every LogQL query starts with a stream selector in braces, and everything after it is a pipeline of stages applied to the matching lines.

``` logql
{job="api", env="prod"} |= "error" | json | status >= 500 | line_format "{{.msg}}"
   stream selector       line filter  parser  label filter    formatter
```

There are two kinds of query. A **log query** returns log lines. A **metric query** wraps a log query in an aggregation and returns a time series, which is what makes a log-based Grafana panel or alert possible.

The deliberate similarity to PromQL is the point: selectors look like PromQL selectors, and the aggregation operators are the same names doing the same things.

------------------------------------------------------------------------

## The Stream Selector Decides The Cost

``` logql
{job="api"}                          # every stream for the job
{job="api", level="error"}           # narrower, cheaper
{job=~"api|worker", env="prod"}      # regex, fully anchored as in PromQL
{job!="noisy-thing"}
```

**The stream selector is mandatory and it is the only part that uses the index.** Everything after it is a scan over the matched chunks. A query selecting one stream over ten minutes reads a few megabytes; the same query with `{env="prod"}` over a week reads everything the cluster produced.

This is the single performance rule in LogQL: **narrow the selector first, and keep the time range small**. Raising query limits to make a broad query succeed is treating the symptom.

------------------------------------------------------------------------

## Line Filters

Line filters operate on the raw log line before any parsing, which makes them the cheapest stage. Loki optimises them heavily.

``` logql
{job="api"} |= "error"                    # contains
{job="api"} != "healthcheck"              # does not contain
{job="api"} |~ "timeout|deadline"         # regex match
{job="api"} !~ "^DEBUG"                   # regex does not match
{job="api"} |= "error" != "expected"      # chained, applied in order
```

**Put line filters immediately after the selector, before any parser.** Parsing is expensive per line; a line filter that discards 99 percent of lines first means the parser runs on one percent of the data. The difference between these two is often an order of magnitude:

``` logql
{job="api"} | json | level="error"        # parses every line, then filters
{job="api"} |= "error" | json | level="error"   # filters cheaply first
```

**Prefer `|=` over `|~` where possible.** A substring match is far faster than a regex, and Loki can use it to skip whole chunks.

------------------------------------------------------------------------

## Parsers

A parser extracts labels from the line body. Those labels exist only for the rest of the query; they never become stream labels and never affect storage.

``` logql
# JSON: every top-level field becomes a label
{job="api"} | json

# JSON: only named fields, with nested access
{job="api"} | json status="response.status", user="request.user_id"

# logfmt: key=value pairs
{job="api"} | logfmt

# regex with named capture groups
{job="nginx"} | regexp `(?P<method>\w+) (?P<path>\S+) (?P<status>\d{3})`

# pattern: simpler and much faster than regexp for fixed-shape lines
{job="nginx"} | pattern `<ip> - - <_> "<method> <path> <_>" <status> <size>`

# unpack: undo the collector's JSON wrapping, restoring original labels
{job="api"} | unpack
```

**`pattern` is the one to reach for on structured-but-not-JSON lines.** `<name>` captures a field, `<_>` skips one, and it is several times faster than the equivalent regex because there is no backtracking.

A parse failure adds a `__error__` label rather than dropping the line, which is why a mixed-format stream produces confusing results. Filter them explicitly:

``` logql
{job="api"} | json | __error__=""          # only lines that parsed
{job="api"} | json | __error__!=""         # only the failures, to see what is odd
```

------------------------------------------------------------------------

## Label Filters And Formatting

Once a parser has produced labels, they can be filtered with comparisons. Unlike stream selectors, these understand numbers and durations.

``` logql
{job="api"} | json | status >= 500
{job="api"} | json | duration > 250ms
{job="api"} | json | bytes > 10MB
{job="api"} | json | status >= 500 and method = "POST"
{job="api"} | json | status >= 500 or duration > 1s
```

Numeric and duration comparison happens only after a parser has run. Comparing against a stream label, which is always a string, silently does a string comparison instead.

``` logql
# Rewrite the displayed line
{job="api"} | json | line_format "{{.method}} {{.path}} -> {{.status}} in {{.duration}}"

# Add or rewrite a label
{job="api"} | json | label_format route=`{ regexReplaceAll "/[0-9]+" .path "/:id" }`

# Drop noisy extracted labels
{job="api"} | json | drop user_agent, referrer

# Keep only what is needed
{job="api"} | json | keep status, route, duration
```

`line_format` and `label_format` are Go templates. `label_format` with `regexReplaceAll` is the standard way to collapse a high-cardinality path into a route template before aggregating on it, which matters a great deal for the metric queries below.

------------------------------------------------------------------------

## Metric Queries

Wrapping a log query in a range aggregation produces a time series, and this is where LogQL earns its place: it turns logs that nobody instrumented into metrics, retroactively.

### Log Range Aggregations

``` logql
# Lines per second per stream
rate({job="api"} |= "error" [5m])

# Count of lines in the window
count_over_time({job="api"} |= "error" [5m])

# Bytes per second
bytes_rate({job="api"}[5m])

# Aggregate over an extracted numeric field
avg_over_time({job="api"} | json | unwrap duration [5m])
quantile_over_time(0.99, {job="api"} | json | unwrap duration [5m])
max_over_time({job="api"} | json | unwrap duration [5m])
sum_over_time({job="api"} | json | unwrap bytes [5m])
```

`unwrap` is what makes the second group work: it takes an extracted label and treats its value as the sample value rather than counting lines. `unwrap duration` on a field like `1.5s` handles the unit; `unwrap_duration` and `unwrap_bytes` exist for explicit conversion.

### Vector Aggregations

The outer layer is the familiar PromQL set: `sum`, `avg`, `min`, `max`, `count`, `stddev`, `stdvar`, `bottomk`, `topk`, `sort`.

``` logql
# Error rate by route
sum by (route) (
  rate({job="api"} |= "error" | json | __error__="" [5m])
)

# The error ratio, entirely from logs
sum(rate({job="api"} | json | status >= 500 [5m]))
  /
sum(rate({job="api"} | json [5m]))

# p99 latency from logs, for a service nobody instrumented
quantile_over_time(0.99, {job="api"} | json | unwrap duration [5m]) by (route)

# The ten noisiest containers by log volume
topk(10, sum by (container) (bytes_rate({namespace="prod"}[5m])))
```

That last one is worth keeping. Log cost is driven by a small number of chatty services, and this finds them in one query.

**The same aggregate-the-rate rule from PromQL applies**: `rate()` goes around the log selector, and `sum` wraps it.

------------------------------------------------------------------------

## Alerting On Logs

Loki has a ruler that evaluates alerting rules in the same format Prometheus uses, sending to the same Alertmanager.

``` yaml
groups:
  - name: logs
    interval: 1m
    rules:
      - alert: ErrorLogSpike
        expr: |
          sum by (job) (rate({env="prod"} |= "error" [5m])) > 10
        for: 10m
        labels: {severity: ticket}
        annotations:
          summary: "{{ $labels.job }} logging {{ $value | printf \"%.1f\" }} errors/sec"

      - alert: PanicLogged
        expr: |
          sum by (job) (count_over_time({env="prod"} |~ "panic:|FATAL" [5m])) > 0
        for: 0m
        labels: {severity: page}
        annotations:
          summary: "{{ $labels.job }} logged a panic"

      - alert: BackupJobSilent
        expr: |
          sum(count_over_time({job="backup"} |= "completed successfully" [26h])) == 0
        for: 0m
        labels: {severity: ticket}
        annotations:
          summary: "No successful backup line in 26 hours"
```

The third pattern, alerting on the **absence** of an expected log line, is one of the most useful things in the whole stack and has no clean equivalent in metrics without instrumenting the job.

**Log-based alerts are second choice where a metric exists.** They cost more to evaluate, they break when somebody rewords a log message, and they depend on the log pipeline being healthy. Use them for what metrics cannot see: panics, specific error strings, and jobs that report success only in prose.

------------------------------------------------------------------------

## Recording Rules

The Loki ruler can also write results back to a Prometheus-compatible store via remote write, which converts an expensive recurring log query into a cheap metric.

``` yaml
groups:
  - name: log-derived
    interval: 1m
    rules:
      - record: job:log_errors:rate5m
        expr: sum by (job) (rate({env="prod"} |= "error" [5m]))
```

This is the right answer for any log query that ends up on a dashboard people leave open. Each refresh of that panel otherwise rescans the chunks.

------------------------------------------------------------------------

## Performance Checklist

In the order that matters:

1.  **Narrow the stream selector.** Most expensive queries are a selector problem.
2.  **Shorten the time range.** Cost is linear in it.
3.  **Line filter before parsing.** `|= "error" | json` beats `| json | level="error"`.
4.  **Prefer `|=` to `|~`**, and `pattern` to `regexp`.
5.  **Drop labels you do not need** before aggregating.
6.  **Record the query** if it runs on a schedule or sits on a dashboard.

Loki parallelises by splitting a query across time and across streams, so a query that cannot be split, typically because it selects a single enormous stream, will not get faster by adding queriers.

------------------------------------------------------------------------

## Quick Reference

| Want | Query |
|----|----|
| Tail one service | `{job="api"}` |
| Errors only | `{job="api"} \|= "error"` |
| Parsed JSON, 5xx only | `{job="api"} \|= "error" \| json \| status >= 500` |
| Error rate by route | `sum by (route) (rate({job="api"} \|= "error" \| json [5m]))` |
| p99 from a duration field | `quantile_over_time(0.99, {job="api"} \| json \| unwrap duration [5m])` |
| Noisiest containers | `topk(10, sum by (container) (bytes_rate({namespace="prod"}[5m])))` |
| Lines that failed to parse | `{job="api"} \| json \| __error__!=""` |
| Nothing logged in 26h | `sum(count_over_time({job="backup"} \|= "ok" [26h])) == 0` |

------------------------------------------------------------------------

## Where Next

- [Loki](../Observability/loki.llms.md) for the storage model these queries run against.
- [PromQL](../Observability/promql.llms.md) for the language this one is modelled on.
- [Tempo](../Observability/tempo.llms.md) for following a trace ID out of a log line.

------------------------------------------------------------------------

Back to top
