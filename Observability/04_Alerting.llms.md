# Alerting

Alerting rules, how Alertmanager groups, inhibits, silences and routes what they produce, and the same job done inside Grafana.

Author

Benedict Thekkel

## The Split

Prometheus decides **whether** something is wrong. Alertmanager decides **who hears about it and how**. They are separate processes with separate configs, and keeping the responsibilities straight is most of what makes an alerting setup comprehensible.

      rule files            Prometheus evaluates every                Alertmanager
      (PromQL + for)  --->  evaluation_interval, fires   --->  group -> inhibit -> silence -> route -> notify
                            alerts as long as true              (dedupe across HA Prometheus pairs)

A firing alert is re-sent to Alertmanager on every evaluation. Alertmanager deduplicates, which is also why a pair of identical Prometheus servers can both send the same alert without paging twice.

------------------------------------------------------------------------

## Alerting Rules

``` yaml
groups:
  - name: api
    interval: 30s
    rules:
      - alert: HighErrorRate
        expr: |
          sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
            /
          sum by (job) (rate(http_requests_total[5m]))
            > 0.05
        for: 10m
        labels:
          severity: page
          team: platform
        annotations:
          summary: "{{ $labels.job }} is returning {{ $value | humanizePercentage }} errors"
          description: >
            Error ratio has been above 5 percent for 10 minutes.
            Check recent deploys and the upstream dependencies.
          runbook_url: https://github.com/bthek1/Knowledge/issues/19
```

| Field | Role |
|----|----|
| `alert` | The name. Becomes the `alertname` label, which is the primary grouping key |
| `expr` | Any PromQL. The alert fires for **each series** the expression returns |
| `for` | How long the condition must hold continuously before firing |
| `labels` | Attached to the alert. This is what routing matches on |
| `annotations` | Human-facing text. Templated, never used for routing |

### `for` Is The Flap Filter

Without `for`, an alert fires the instant the expression returns anything, including on a single scrape blip. With `for: 10m` the alert enters the `pending` state, and only becomes `firing` if it is still true 10 minutes later.

The tension is real: a long `for` means a slow page, a short one means noise. The usual shape is a short `for` on a severe, unambiguous condition and a longer one on a ratio that can spike briefly. Burn-rate alerting, covered in [SLOs and alerting practice](../Observability/slos_and_alerting_practice.llms.md), is the principled answer, using a fast window and a slow window together instead of a single arbitrary duration.

**`for` restarts from zero whenever the expression stops returning that series.** A metric that disappears intermittently, rather than going below the threshold, resets the timer, and the alert never fires. This is the most common reason a condition that is visibly true on a dashboard never pages.

### One Alert Per Returned Series

`expr` returning ten series produces ten alerts, each carrying that series’ labels. This is usually what you want, and occasionally a paging storm: an expression grouped by `instance` across a 200-node fleet during a network partition sends 200 alerts. Either aggregate the expression to a level where one alert is the right number, or rely on Alertmanager grouping, below.

### Templating

Annotations are Go templates with `$labels`, `$value` and a set of formatting helpers.

``` yaml
summary: "{{ $labels.instance }} disk {{ $labels.mountpoint }} at {{ $value | humanizePercentage }}"
description: "Free space {{ with printf `node_filesystem_avail_bytes{instance='%s'}` $labels.instance | query }}{{ . | first | value | humanize1024 }}B{{ end }}"
```

`humanize`, `humanize1024`, `humanizeDuration`, `humanizePercentage` and `humanizeTimestamp` cover most needs. Templates are evaluated at notification time, and a template error suppresses the notification, so keep them simple. Anything requiring a `query` call inside a template is usually better solved by putting the extra context in a label on the expression itself.

### Rules Worth Having Everywhere

``` yaml
      - alert: TargetDown
        expr: up == 0
        for: 5m
        labels: {severity: page}
        annotations:
          summary: "{{ $labels.job }} target {{ $labels.instance }} is down"

      - alert: DiskWillFill
        expr: |
          predict_linear(node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}[6h], 24*3600) < 0
          and node_filesystem_avail_bytes / node_filesystem_size_bytes < 0.30
        for: 1h
        labels: {severity: ticket}
        annotations:
          summary: "{{ $labels.instance }}:{{ $labels.mountpoint }} fills within 24h"

      - alert: PrometheusRuleFailures
        expr: increase(prometheus_rule_evaluation_failures_total[10m]) > 0
        for: 0m
        labels: {severity: page}
        annotations:
          summary: "Prometheus rule evaluation is failing: alerting is degraded"
```

The third is the one people forget. **Monitor the monitoring.** A Prometheus whose rules are failing to evaluate is silently not alerting on anything, and nothing else in the system will tell you.

### Validating Rules

``` bash
promtool check rules /etc/prometheus/rules/*.yml
promtool test rules /etc/prometheus/tests/*.yml
```

`promtool test rules` is a genuine unit-test runner for alerts: it takes a synthetic series definition and asserts which alerts fire and when.

``` yaml
rule_files: [../rules/api.yml]
evaluation_interval: 1m
tests:
  - interval: 1m
    input_series:
      - series: 'http_requests_total{job="api", status="500"}'
        values: '0+10x20'            # 10 per minute for 20 minutes
      - series: 'http_requests_total{job="api", status="200"}'
        values: '0+90x20'            # 90 per minute, so a 10 percent error ratio
    alert_rule_test:
      - eval_time: 15m
        alertname: HighErrorRate
        exp_alerts:
          - exp_labels: {job: api, severity: page, team: platform}
```

Alerts are the part of a stack that only runs during an incident, which is the worst possible time to discover a typo in a label selector. Test the ones that page.

------------------------------------------------------------------------

## Alertmanager

``` yaml
global:
  resolve_timeout: 5m

route:
  receiver: default                       # the fallback
  group_by: [alertname, cluster, service]
  group_wait: 30s                         # hold a new group briefly to collect siblings
  group_interval: 5m                      # wait before sending an updated group
  repeat_interval: 4h                     # re-notify an unresolved group this often
  routes:
    - matchers: [severity = page]
      receiver: pager
      group_wait: 10s
      continue: false                     # stop here, do not fall through

    - matchers: [severity = ticket]
      receiver: email

    - matchers: [team = platform, severity =~ "warning|info"]
      receiver: platform-chat

inhibit_rules:
  - source_matchers: [alertname = TargetDown]
    target_matchers: [severity =~ "warning|info"]
    equal: [instance]                     # same instance only

receivers:
  - name: default
    webhook_configs:
      - url: http://localhost:5001/

  - name: pager
    webhook_configs:
      - url: https://events.pagerduty.com/v2/enqueue
        send_resolved: true

  - name: email
    email_configs:
      - to: ops@example.com
        from: alertmanager@example.com
        smarthost: smtp.example.com:587
        auth_username: alertmanager
        auth_password_file: /etc/alertmanager/smtp_password

  - name: platform-chat
    slack_configs:
      - api_url_file: /etc/alertmanager/slack_url
        channel: "#platform-alerts"
        send_resolved: true
```

### The Route Tree

Routing is a tree walked depth first. An alert enters at the root, and at each node it takes the **first** matching child. `continue: true` makes it keep evaluating siblings after a match, which is how one alert reaches two receivers.

A node that matches but has no matching children uses its own receiver. A node inherits `group_by`, the timers and the receiver from its parent unless it overrides them, which is why the root needs a sane default and why most real trees are shallow.

### The Four Timers

These are the part people get wrong, and the symptom is either a flood or a suspicious silence.

| Timer | Meaning | Typical |
|----|----|----|
| `group_wait` | After the first alert in a new group, wait this long for siblings before the first notification | 30 s, or 10 s for pages |
| `group_interval` | Minimum gap before sending an update about a group that has changed | 5 m |
| `repeat_interval` | Re-send an unchanged, still-firing group after this long | 4 h |
| `resolve_timeout` | Treat an alert as resolved if Prometheus stops re-sending it for this long | 5 m |

`group_wait` exists so that a rack losing power produces one notification listing twelve hosts rather than twelve notifications. `repeat_interval` set too short is how an on-call rotation learns to ignore the alerting channel.

### Grouping

`group_by` decides what counts as one notification. The two extremes are both bad: `group_by: [...]` (the literal special value) disables grouping entirely and sends everything separately, while `group_by: [alertname]` across a whole fleet collapses unrelated clusters into one message.

The workable default is `[alertname, cluster, service]`: alerts of the same kind, in the same place, about the same thing, arrive together.

### Inhibition

An inhibit rule suppresses one alert while another is firing. The classic case is a node going down, which also trips every service-level alert on that node.

``` yaml
inhibit_rules:
  - source_matchers: [severity = critical]
    target_matchers: [severity = warning]
    equal: [alertname, cluster, service]
```

`equal` is the crucial part and is frequently too broad. It lists the labels that must **match between the two alerts** for suppression to apply. Omitting it means any critical alert anywhere suppresses every warning everywhere, which turns a helpful feature into a way to miss an unrelated incident.

### Silences

A silence is a time-bounded mute created through the UI or API, matching on labels. It is for planned work: a maintenance window, a known-broken thing already being worked on.

``` bash
amtool silence add alertname=HighErrorRate job=api \
  --duration=2h --comment="deploying fix, issue #19" --author=ben

amtool silence query
amtool silence expire <id>
```

`amtool` is also the fastest way to debug a route without waiting for a real alert:

``` bash
# Which receiver would this alert reach
amtool config routes test --config.file=alertmanager.yml \
  severity=page team=platform alertname=HighErrorRate

# Render the whole tree
amtool config routes show --config.file=alertmanager.yml
```

### High Availability

Alertmanager clusters by gossip, and the cluster’s job is deduplication, not load sharing. Every instance receives every alert; they coordinate so exactly one notification goes out.

``` bash
alertmanager --cluster.peer=am-2:9094 --cluster.peer=am-3:9094
```

The matching pattern on the Prometheus side is to run two identical Prometheus servers both scraping everything and both sending to all Alertmanagers. Duplicate alerts are expected and are exactly what the dedupe handles.

------------------------------------------------------------------------

## Grafana Unified Alerting

Grafana has its own alerting engine, which can evaluate rules against any datasource and can use Grafana’s own notification routing or hand off to an external Alertmanager.

|  | Prometheus plus Alertmanager | Grafana unified alerting |
|----|----|----|
| Rule storage | Files on disk, version controlled naturally | Grafana database, or provisioned from files |
| Datasources | Prometheus only | Any: Loki, SQL, Graphite, mixed in one rule |
| Evaluation | In Prometheus, close to the data | In Grafana, pulling from the datasource |
| Config as code | Native | Possible, via provisioning files or Terraform |
| Survives Grafana being down | Yes | No |
| Multi-datasource conditions | No | Yes, several queries plus expressions in one rule |

**The argument for Prometheus-side rules** is independence. Alerting continues when Grafana is down, rules live in Git next to the code, and evaluation happens next to the data with no network hop.

**The argument for Grafana-side rules** is reach. Alerting on a Loki log pattern, or on a condition combining a Prometheus metric with a row count from Postgres, is not expressible in a Prometheus rule at all.

The common arrangement is both: infrastructure and service alerts as Prometheus rules, log-based and cross-datasource alerts in Grafana, and a single Alertmanager receiving from both so there is one route tree and one silence list. Grafana can be pointed at an external Alertmanager precisely to allow this.

Grafana alert rules can be provisioned from YAML rather than clicked, which is the only sane way to run them. That belongs with the rest of the provisioning story in [dashboards as code](../Observability/dashboards_as_code.llms.md).

------------------------------------------------------------------------

## What Makes An Alert Worth Having

The test that survives contact with an on-call rotation: **every page must be urgent, actionable, and about a symptom.**

- **Urgent.** If it can wait until Monday, it is a ticket, not a page. Route it accordingly rather than paging and hoping the recipient exercises judgement at 3 a.m.
- **Actionable.** There must be something the recipient can do. An alert whose runbook says “watch it” trains people to ignore alerts.
- **A symptom, not a cause.** Alert on the error ratio users experience, not on CPU being high. High CPU that harms nobody is not an incident, and the cause-based alert will miss every outage that has a different cause.

Cause-based signals still belong in dashboards; they are what you look at after the symptom alert fires. The failure mode of cause-based alerting is a rotation drowning in notifications about conditions that never affected anyone, and it ends the same way every time: the channel gets muted, and the one alert that mattered is muted with it.

------------------------------------------------------------------------

## Where Next

- [SLOs and alerting practice](../Observability/slos_and_alerting_practice.llms.md) for burn rates and error budgets, the discipline behind the rules above.
- [PromQL](../Observability/promql.llms.md) for the expressions themselves.
- [Grafana](../Observability/grafana.llms.md) for the read side.

------------------------------------------------------------------------

Back to top
