# PromQL

The query language: selectors, rate versus increase, counter resets, aggregation, joins, recording rules, and the traps that produce confidently wrong numbers.

Author

Benedict Thekkel

## The Four Expression Types

Every PromQL expression evaluates to one of four things, and most errors come from using one where another is required.

| Type | Looks like | Is |
|----|----|----|
| Instant vector | `node_cpu_seconds_total` | One sample per matching series, at the evaluation time |
| Range vector | `node_cpu_seconds_total[5m]` | A window of samples per series. Only useful as a function argument |
| Scalar | `0.95`, `time()` | A single number |
| String | `"literal"` | Rarely used |

The rule that follows: **a range vector cannot be graphed or compared**. `node_cpu_seconds_total[5m] > 0.5` is a parse error, and `rate(...)` exists precisely to collapse a range vector back into an instant vector. Any time PromQL complains about an unexpected type, the cause is almost always a range selector that never got wrapped in a function.

------------------------------------------------------------------------

## Selectors

``` promql
# Everything with this name
http_requests_total

# Exact label match
http_requests_total{job="api", status="500"}

# Regex match, anchored at both ends automatically
http_requests_total{status=~"5.."}

# Negation, and negative regex
http_requests_total{status!="200"}
http_requests_total{route!~"/health|/metrics"}

# The name is a label too
{__name__=~"node_cpu.*", mode="idle"}
```

Four operators: `=`, `!=`, `=~`, `!~`. Regexes are RE2 and are **fully anchored**, so `status=~"5"` matches only the literal string `5`, not `500`. Write `status=~"5.*"` or `status=~"5.."`.

**A selector must match on something other than an empty string.** `{job=~".*"}` is rejected because it would match every series in the database. `{__name__=~".+"}` works and is the usual way to ask a cardinality question deliberately.

### Offset And @

``` promql
# The same query as it stood a week ago
sum(rate(http_requests_total[5m] offset 7d))

# Pinned to an absolute timestamp, regardless of the graph range
sum(rate(http_requests_total[5m] @ 1758585600))

# Week-over-week ratio
sum(rate(http_requests_total[5m])) / sum(rate(http_requests_total[5m] offset 7d))
```

`offset` shifts the lookback. `@` pins to an absolute time, which is what makes it possible to compare a moving window against a fixed reference point inside one expression.

------------------------------------------------------------------------

## rate, irate, increase

This is the centre of the language and the source of most wrong answers.

A counter only goes up, so its raw value is uninteresting. All three of these functions answer “how fast is it going up”, differently.

``` promql
# Per-second average over the window. The default choice, always.
rate(http_requests_total[5m])

# Total over the window, extrapolated. rate() * window, roughly.
increase(http_requests_total[5m])

# Per-second, computed from only the LAST TWO samples in the window.
irate(http_requests_total[5m])
```

### All Three Handle Counter Resets

When a process restarts, its counters go back to zero. A naive derivative would produce a large negative spike. `rate`, `irate`, `increase` and `resets` all detect a decrease between consecutive samples, treat it as a reset, and add the pre-reset value back in.

This is why **you must never compute a rate by hand**. `(x - x offset 5m) / 300` looks equivalent and is wrong on every restart, and restarts are exactly when you are looking at the graph.

### Use rate, Not irate

`irate` uses only the last two samples in the range, so it reacts instantly and is extremely spiky. It is for zooming into a short incident, never for alerting or for a dashboard covering hours, where it will alias badly: with a 15 s scrape and a 1 h graph the renderer picks one pair of samples per pixel and throws away everything between them, so brief spikes appear or vanish depending on the window width.

`rate` fits a line across the whole window and is the right default in essentially every case.

### The Window Must Cover At Least Two Scrapes

`rate()` needs two samples to compute anything, so the range has to span at least two scrape intervals. Prometheus 3 additionally requires a bit of margin.

**The practical rule is a window of at least 4x the scrape interval.** At 15 s scrapes, `[1m]` is the minimum and `[5m]` is the comfortable default. `rate(x[30s])` on a 15 s scrape returns empty whenever a single scrape is late, which makes an alert that silently stops evaluating: no data, no firing, no error.

### increase Extrapolates, So It Lies About Integers

`increase()` is defined as the rate over the window multiplied by the window length, with extrapolation to the range boundaries. The result is a float, and it is frequently not the integer you expect.

``` promql
increase(http_requests_total[1h])   # can return 5.999999 or 6.0000001 for 6 events
```

For a low-frequency counter this is visibly wrong, and `increase(errors_total[1h]) > 5` can fire on five actual errors. Where the exact count matters, alert on `rate` over a longer window instead, or accept the approximation knowingly. This is a documented consequence of the extrapolation, not a bug.

------------------------------------------------------------------------

## Aggregation

``` promql
# Collapse everything to one number
sum(rate(http_requests_total[5m]))

# Keep a dimension: one result per job
sum by (job) (rate(http_requests_total[5m]))

# Collapse only one dimension, keep the rest
sum without (instance) (rate(http_requests_total[5m]))
```

Operators: `sum`, `min`, `max`, `avg`, `count`, `stddev`, `stdvar`, `topk`, `bottomk`, `quantile`, `group`, `count_values`.

**`by` keeps only the listed labels. `without` drops only the listed ones.** Prefer `without` when adding a new label to instrumentation should not silently change the query, and `by` when the intended grouping is fixed. A dashboard built with `by (job)` keeps working when someone adds a `version` label; one built with `without (instance)` starts splitting every line in two.

### Aggregate Rate, Do Not Rate Aggregate

``` promql
sum(rate(http_requests_total[5m]))      # correct
rate(sum(http_requests_total)[5m:])     # wrong
```

The second form sums the counters first. When one instance restarts, the sum drops, and the reset detection cannot tell that drop apart from a genuine decrease because it is now looking at an aggregate of many counters rather than one counter. The rule: **`rate()` goes on the innermost expression, directly around the counter selector**, and aggregation wraps it.

### Averaging Averages

``` promql
avg(rate(http_request_duration_seconds_sum[5m]) / rate(http_request_duration_seconds_count[5m]))   # wrong
sum(rate(http_request_duration_seconds_sum[5m])) / sum(rate(http_request_duration_seconds_count[5m]))   # right
```

The first takes the mean of per-instance means, which weights a barely-used instance equally with a busy one. The second is the true fleet-wide mean because it divides total time by total count. The same reasoning applies to any ratio: **sum the numerators and sum the denominators, then divide**.

------------------------------------------------------------------------

## Histograms And Quantiles

``` promql
# p99 latency across the fleet
histogram_quantile(
  0.99,
  sum by (le) (rate(http_request_duration_seconds_bucket[5m]))
)

# p99 per route
histogram_quantile(
  0.99,
  sum by (le, route) (rate(http_request_duration_seconds_bucket[5m]))
)
```

Three things must be true or the answer is garbage.

1.  **`rate()` around the bucket counter**, because buckets are counters.
2.  **`le` must survive the aggregation.** `sum by (le)` or `sum without (instance)`. Dropping `le` leaves nothing to interpolate over and returns `NaN`.
3.  **Aggregate before `histogram_quantile`, never after.** Taking the average of per-instance p99s is meaningless; quantiles do not average.

**The result is only as good as the bucket boundaries.** `histogram_quantile` interpolates linearly inside the bucket the quantile falls into. If the buckets jump from 1 s to 10 s and the true p99 is 2 s, the answer will be somewhere near 1.9 s by pure luck of the interpolation. When a quantile sits inside the last finite bucket it is at least bounded; when it falls into `+Inf` the function returns the lower bound of that bucket and the true value is unknowable.

Sanity check a suspicious quantile against `_sum / _count`. If the mean is 400 ms and the reported p99 is 410 ms, the buckets are too coarse to resolve the tail.

------------------------------------------------------------------------

## Vector Matching And Joins

Binary operations match series on their full label sets. Identical labels on both sides just work.

``` promql
# Both sides have the same labels: element-wise division
node_filesystem_avail_bytes / node_filesystem_size_bytes
```

When the label sets differ, the match must be specified.

``` promql
# Many-to-one: many pod series, one info series per pod
sum by (pod) (rate(container_cpu_usage_seconds_total[5m]))
  * on (pod) group_left (owner_name)
  kube_pod_owner
```

`on (labels)` restricts matching to the listed labels. `ignoring (labels)` matches on everything except those. `group_left` means the left side has the many and pulls extra labels from the right; `group_right` is the mirror image.

**The info-metric join is the pattern worth memorising.** Exporters commonly emit a `*_info` metric with value 1 whose only job is to carry metadata labels, and this is how that metadata gets attached to a real measurement.

``` promql
# Attach the version label from an info metric onto a rate
sum by (instance) (rate(app_requests_total[5m]))
  * on (instance) group_left (version)
  app_build_info
```

Multiplying by a metric whose value is always 1 leaves the numbers unchanged and copies the labels across. That is the whole trick.

**`many-to-many matching not allowed` means a `group_left` is missing** or the `on` clause is not specific enough to make one side unique. The fix is almost never to add more labels to `on`; it is to work out which side is the “one”.

------------------------------------------------------------------------

## Absence, Staleness And Missing Data

The hardest class of alert is on something that stopped happening, because a series that does not exist cannot be compared to a threshold.

``` promql
# Fires when the series is entirely absent
absent(up{job="api"})

# Absent, per label combination, from a range
absent_over_time(up{job="api"}[10m])

# Has the counter moved at all in an hour
increase(backup_success_total[24h]) == 0

# Time since the last successful run
time() - backup_last_success_timestamp_seconds > 86400
```

**`absent()` only fires when the selector matches nothing at all.** It cannot tell you that one of twenty instances vanished, because the other nineteen still match. For per-instance disappearance the usual pattern is a comparison against what should exist, via `kube_*` object metrics or a static list.

**The staleness window is five minutes.** A series whose target disappears keeps being returned by instant queries for up to five minutes, then stops. An alert with `for: 2m` on an absence condition may therefore not fire until well after the fact. Prometheus writes explicit stale markers when it knows a target went away, which shortens this, but not for every disappearance path.

**A timestamp gauge is the most reliable staleness signal.** `time() - last_success_timestamp_seconds` always has a value as long as the exporter is up, degrades gracefully, and says exactly how late the thing is.

------------------------------------------------------------------------

## Subqueries

A subquery evaluates an instant-vector expression over a range, producing a range vector. The syntax is `[range:resolution]`.

``` promql
# The maximum 5m request rate seen in the last hour
max_over_time(rate(http_requests_total[5m])[1h:1m])

# Was the error ratio ever above 5 percent today
max_over_time(
  (sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])))[24h:5m]
) > 0.05
```

They are expensive: a `[24h:5m]` subquery evaluates the inner expression 288 times per outer evaluation. Useful for exploration, and a strong signal that a recording rule should exist. Never put one in an alerting rule that evaluates every 15 seconds.

------------------------------------------------------------------------

## Recording Rules

A recording rule evaluates an expression on a schedule and stores the result as a new series. It exists for two reasons: to make a dashboard fast, and to make an expensive expression cheap enough to alert on.

``` yaml
groups:
  - name: api-slo
    interval: 30s
    rules:
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))

      - record: job:http_errors:rate5m
        expr: sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))

      - record: job:http_error_ratio:rate5m
        expr: job:http_errors:rate5m / job:http_requests:rate5m
```

**The naming convention is `level:metric:operations`.** The prefix is the aggregation level (the labels that survive), the middle is the metric, the suffix describes what was done. `job:http_requests:rate5m` reads as “summed to job level, of http_requests, as a 5-minute rate”. It is a convention rather than a rule, and it is worth following because it makes the aggregation level obvious at a glance in a dashboard nobody has opened for a year.

**Rules within a group evaluate sequentially**, so a rule can depend on one defined above it in the same group, as the third rule above does. Rules in different groups evaluate concurrently and must not depend on each other.

**Do not record a rule whose expression is cheap.** Every recording rule is a new series written forever. The point is to precompute the expensive aggregation once instead of on every dashboard refresh, not to alias every query.

------------------------------------------------------------------------

## Reading The Cost Of A Query

``` promql
# The 10 metric names with the most series
topk(10, count by (__name__) ({__name__=~".+"}))

# Series per job
sort_desc(count by (job) ({__name__=~".+"}))

# Which label is exploding, for one metric
count(count by (pod) (container_cpu_usage_seconds_total))

# Scrape health across the fleet
sum by (job) (up) / count by (job) (up)

# Samples ingested per second, the ingestion budget
rate(prometheus_tsdb_head_samples_appended_total[5m])
```

`/tsdb-status` in the web UI gives the same top-cardinality view without writing a query, and Prometheus logs a slow-query warning for anything above `--query.timeout`. A query touching more than a few million samples will be slow no matter how it is written; the answer is a recording rule or fewer series, not a cleverer expression.

------------------------------------------------------------------------

## Quick Reference: The Traps

| Symptom | Cause |
|----|----|
| Empty result from `rate()` | Range narrower than 2 scrape intervals. Use at least 4x |
| Negative spike on a graph | A hand-rolled derivative instead of `rate()`. Counter reset |
| `histogram_quantile` returns `NaN` | `le` was dropped by the aggregation |
| p99 equals the top bucket bound | The tail is past the last finite bucket, value unknowable |
| Latency graph looks too good | Averaged per-instance quantiles, or averaged averages |
| `many-to-many matching not allowed` | Missing `group_left` or `group_right` |
| Alert never fires on an outage | Condition needs a series that disappears with the target. Use `absent()` or a timestamp gauge |
| `increase()` returns 5.9999 | Extrapolation. Expected behaviour, not a bug |
| Regex matches nothing | Regexes are fully anchored. `5` is not `5..` |
| Dashboard slow after adding a label | Cardinality. Check `/tsdb-status` |

------------------------------------------------------------------------

## Where Next

- [Alerting](../Observability/alerting.llms.md) turns these expressions into rules and routes what they produce.
- [LogQL](../Observability/logql.llms.md) is the deliberately similar language for logs.
- [SLOs and alerting practice](../Observability/slos_and_alerting_practice.llms.md) covers burn rate queries, which are the most demanding PromQL most people write.

------------------------------------------------------------------------

Back to top
