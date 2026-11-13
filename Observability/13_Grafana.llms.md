# Grafana

The read side: datasources, Explore, the correlations between signals, variables and transformations, and how to build a dashboard somebody will actually use.

Author

Benedict Thekkel

## What Grafana Is Doing

Grafana stores no telemetry. It is a query front end that holds datasource definitions, dashboard JSON, alert rules, and user and permission state, and it issues queries to the real backends on every panel refresh.

This is worth stating because it explains most of Grafana’s behaviour. A slow dashboard is almost never a Grafana problem, it is a query problem. A panel showing no data is usually a datasource or query problem. And the same dashboard pointed at a different Prometheus works without modification, which is what makes provisioning and templating possible at all.

------------------------------------------------------------------------

## Installing And Running

``` yaml
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD__FILE=/run/secrets/grafana_admin
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_SERVER_ROOT_URL=http://192.168.2.205:3000
      - GF_FEATURE_TOGGLES_ENABLE=traceqlEditor
    volumes:
      - grafana-data:/var/lib/grafana
      - ./provisioning:/etc/grafana/provisioning:ro
      - ./dashboards:/var/lib/grafana/dashboards:ro
    secrets:
      - grafana_admin
    restart: unless-stopped
```

On a host rather than a container:

``` sh
sudo apt-get install -y adduser libfontconfig1 musl
wget https://dl.grafana.com/oss/release/grafana_11.3.0_amd64.deb
sudo dpkg -i grafana_11.3.0_amd64.deb
sudo systemctl enable --now grafana-server
sudo systemctl status grafana-server
```

It lands on port 3000, first login `admin` / `admin`, and it demands a password change immediately. **Change the default and do not leave it reachable from outside the LAN**; a Grafana with default credentials is a read path into every metric, log and trace in the stack, and it is a standard target.

**Every environment variable maps to a config key** by the pattern `GF_<SECTION>_<KEY>`, so `GF_SECURITY_ADMIN_PASSWORD` is `[security] admin_password`. Any of them can take a `__FILE` suffix to read from a file instead, which is how secrets get in without appearing in `docker inspect`.

The `/var/lib/grafana` volume holds `grafana.db`, the SQLite database with dashboards, users and alert state. **It is the only stateful thing here**, and it is what needs backing up. A provisioned-everything setup makes it nearly disposable, which is the argument for [dashboards as code](../Observability/dashboards_as_code.llms.md).

------------------------------------------------------------------------

## Datasources

A datasource is a backend plus its access settings. Grafana proxies queries server-side by default, so the browser never talks to Prometheus directly and the backend does not need to be reachable from the client network.

The ones this stack uses: Prometheus, Loki, Tempo, Pyroscope, and usually a SQL datasource or two. Mixed datasource panels are possible, and the `-- Mixed --` datasource lets one panel query several backends at once.

**Configure datasources by provisioning, not by clicking.** A hand-created datasource has a UID Grafana generated at random, and every dashboard that references it embeds that UID, so the dashboard is not portable to another Grafana. Provisioning pins the UID. This is the single most common reason an exported dashboard shows no data when imported elsewhere.

------------------------------------------------------------------------

## Explore Versus Dashboards

They are different tools for different moments and conflating them wastes time.

**Explore** is the ad-hoc query view. One query, no saving, fast iteration, and a split view for comparing two queries or two datasources side by side. It is where an investigation happens.

**Dashboards** are the saved, shared, parameterised view. They answer questions somebody already knew to ask.

The useful workflow is that investigations start in Explore, and anything worth asking twice graduates into a dashboard. A dashboard built speculatively, for questions nobody has asked yet, is usually the kind nobody opens.

------------------------------------------------------------------------

## Correlating The Signals

This is the part that justifies running the whole stack, and it is configured on the **datasources**, not in panels.

**Metric to trace, with exemplars.** The Prometheus datasource is configured with an exemplar link naming the Tempo datasource and the label carrying the trace ID. Latency histogram panels then render clickable dots, each a real trace from that bucket at that moment.

``` yaml
    jsonData:
      exemplarTraceIdDestinations:
        - name: trace_id
          datasourceUid: tempo
```

**Trace to logs.** The Tempo datasource is configured with a Loki datasource, which labels to carry across, and a time window. Opening a span then offers “Logs for this span”, running a Loki query scoped to that service and time range.

``` yaml
    jsonData:
      tracesToLogsV2:
        datasourceUid: loki
        spanStartTimeShift: -5m
        spanEndTimeShift: 5m
        tags: [{key: 'service.name', value: 'service'}]
        filterByTraceID: true
```

**Logs to trace, with derived fields.** The Loki datasource extracts a trace ID from log lines with a regex and turns it into a link to Tempo. This is what makes a trace ID in a log line clickable.

``` yaml
    jsonData:
      derivedFields:
        - name: TraceID
          matcherRegex: 'trace_id=(\w+)'
          url: '${__value.raw}'
          datasourceUid: tempo
```

**Trace to profiles.** Tempo configured against Pyroscope links a span to the flame graph for that service and time window.

Set up, the investigation path is: alert fires, dashboard shows the latency spike, click an exemplar to get a real trace, see which span is slow, click through to that span’s logs, and if the time is inside one process rather than waiting on another, click to the profile. Each step is one click, and without this configuration each step is a manual copy-paste of an ID between three tools.

------------------------------------------------------------------------

## Variables

Variables are what make one dashboard serve a fleet instead of one host.

| Type | Produces |
|----|----|
| Query | Values from a datasource query. The main one |
| Custom | A hand-written list |
| Constant | A fixed value, usually hidden |
| Interval | Time windows, often used as `$__interval` alternatives |
| Datasource | Lets the viewer switch backend, for multi-cluster setups |
| Ad hoc filters | Arbitrary label filters applied to every query on the dashboard |
| Text box | Free input |

``` promql
# A query variable listing every job
label_values(up, job)

# Chained: instances, but only for the selected job
label_values(up{job="$job"}, instance)

# Metric names matching a prefix
metrics(node_cpu)
```

``` promql
# Using them, with the multi-value regex format
sum by (instance) (rate(node_cpu_seconds_total{job="$job", instance=~"$instance"}[$__rate_interval]))
```

**`$__rate_interval` rather than a hard-coded `[5m]`.** Grafana computes it from the panel width and the datasource’s scrape interval, guaranteeing the window covers at least four scrapes. This is the built-in fix for the empty-`rate()` trap from [PromQL](../Observability/promql.llms.md), and it also stops a zoomed-out dashboard from aliasing. Use it in every rate query on a dashboard.

**Multi-value variables interpolate as a regex**, so the selector must be `=~` and not `=`. A multi-select variable with `instance="$instance"` silently matches nothing as soon as two values are selected.

**`$__interval` is different from `$__rate_interval`**: it is the time per pixel, appropriate for `_over_time` functions, and too small for `rate()` on a wide time range.

------------------------------------------------------------------------

## Panels Worth Using

**Time series** for anything over time, which is most things. **Stat** for a single current number with an optional sparkline. **Gauge** only when there is a real maximum; a gauge with an arbitrary max is worse than a number. **Bar gauge** for ranked comparison, which is usually better than a pie chart. **Table** for lists, and it pairs with transformations. **State timeline** for discrete states over time, which is excellent for `up` across a fleet. **Heatmap** for histogram distributions over time, which shows the bimodal latency a p99 line hides. **Logs** for Loki output. **Traces** and **Flame graph** for Tempo and Pyroscope.

**Heatmaps deserve more use than they get.** A p99 line says the tail is at 800 ms. A heatmap of the same histogram shows whether that is one slow cohort and one fast cohort, or a uniform spread, and those call for completely different investigations.

------------------------------------------------------------------------

## Transformations

Transformations reshape query results in the browser, after the query and before the panel. They are the answer to “the data is nearly right but not in the shape this panel wants”.

The ones that earn their keep: **Organize fields** (rename, reorder, hide columns), **Join by field** (combine two queries into one table on a shared label), **Add field from calculation** (a ratio between two queries without expressing it in PromQL), **Group by** (aggregate rows), **Filter data by values**, and **Config from query results**, which sets thresholds or display names from a second query.

**Do the work in the query where possible.** A transformation runs in the browser on whatever the query returned, so it cannot reduce what was transferred, and a dashboard that pulls 50,000 series to filter them client-side is slow no matter how elegant the transformation chain is. Transformations are for shaping, not for filtering.

------------------------------------------------------------------------

## Building A Dashboard Somebody Uses

**One dashboard, one question.** “Is the API healthy” is a dashboard. “Everything about Kubernetes” is a directory of panels nobody reads.

**Top row answers the question.** Put the three or four numbers that say healthy or not healthy at the top: request rate, error rate, latency, saturation. Everything below is for the reader who has already decided something is wrong.

**Order by how an investigation proceeds**, symptoms first and causes below, so scrolling down is the same motion as narrowing the problem.

**Set units and thresholds.** A panel reading `0.0234` with no unit is a number nobody can act on. Seconds, bytes, percent, requests per second: set them.

**Use recording rules for anything slow.** A dashboard refreshing a `[24h]` subquery every 30 seconds is a self-inflicted load problem on Prometheus. See [PromQL](../Observability/promql.llms.md).

**Link dashboards to each other.** Data links and dashboard links turn a set of dashboards into a navigable path rather than a menu to hunt through.

**Write the panel description.** The hover text is where “this counts only successful writes” lives, and it is the difference between a panel being trusted and being re-derived by the next person.

------------------------------------------------------------------------

## Access And Sharing

Organisations contain users and teams; folders hold dashboards and carry permissions; roles are Viewer, Editor and Admin, with fine-grained RBAC in Enterprise.

**Service accounts replace the old API keys** for programmatic access, with their own tokens and roles. This is what Terraform, CI and any script should authenticate with.

**Keep real tokens out of the repo.** A Grafana service-account token is a credential like any other, and `just scan` exists to catch exactly this case. Reference them from environment variables or files, and let the documented examples carry obvious placeholders.

**Public dashboards** expose a dashboard without authentication via a generated URL. They run queries as a privileged internal user, so anyone with the link can see the data. That is occasionally what you want and is an easy way to leak an entire metrics backend.

**Snapshots** are different and safer: they embed the current query results as static data, share nothing live, and do not reach the datasource at all.

------------------------------------------------------------------------

## Where Next

- [Dashboards as code](../Observability/dashboards_as_code.llms.md) for provisioning all of the above from files.
- [Alerting](../Observability/alerting.llms.md) for Grafana’s own alerting engine.
- [PromQL](../Observability/promql.llms.md), [LogQL](../Observability/logql.llms.md) and [Tempo](../Observability/tempo.llms.md) for the queries behind the panels.

------------------------------------------------------------------------

Back to top
