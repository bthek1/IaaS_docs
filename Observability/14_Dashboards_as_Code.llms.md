# Dashboards as Code

Provisioning datasources, dashboards and alert rules from files, the Terraform provider, Grizzly, and why clicking a dashboard together is a decision you pay for later.

Author

Benedict Thekkel

## The Problem With Clicking

A dashboard built in the UI lives in Grafana’s SQLite database. That means no review, no history beyond Grafana’s own version list, no way to apply the same change to staging and production, and a restore-from-backup as the only disaster recovery. The dashboard that matters most during an incident is the one nobody can reproduce.

Provisioning moves all of it into files: datasources, dashboards, alert rules, contact points, notification policies. Grafana reads them on startup and on a timer, and what is in the files wins.

The honest tradeoff is that **provisioned objects are read-only in the UI**. People who built dashboards by dragging panels now have to edit JSON, which is genuinely worse for exploration. The workable pattern is to build in the UI, export, and commit, keeping the UI as the editor and the files as the source of truth.

------------------------------------------------------------------------

## File Provisioning

Grafana reads `/etc/grafana/provisioning/` with one subdirectory per object type.

    provisioning/
      datasources/datasources.yaml
      dashboards/dashboards.yaml        # the provider, pointing at a directory
      alerting/rules.yaml
      alerting/contactpoints.yaml
      alerting/policies.yaml
    dashboards/                         # the dashboard JSON itself
      api-overview.json
      node-health.json

### Datasources

``` yaml
apiVersion: 1

datasources:
  - name: Prometheus
    uid: prometheus                 # PIN THIS
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    jsonData:
      timeInterval: 15s             # must match the scrape interval
      httpMethod: POST
      exemplarTraceIdDestinations:
        - name: trace_id
          datasourceUid: tempo

  - name: Loki
    uid: loki
    type: loki
    access: proxy
    url: http://loki:3100
    jsonData:
      maxLines: 1000
      derivedFields:
        - name: TraceID
          matcherRegex: 'trace_id=(\w+)'
          url: '${__value.raw}'
          datasourceUid: tempo

  - name: Tempo
    uid: tempo
    type: tempo
    access: proxy
    url: http://tempo:3200
    jsonData:
      tracesToLogsV2:
        datasourceUid: loki
        spanStartTimeShift: -5m
        spanEndTimeShift: 5m
        filterByTraceID: true
      tracesToProfiles:
        datasourceUid: pyroscope
        profileTypeId: 'process_cpu:cpu:nanoseconds:cpu:nanoseconds'
      serviceMap:
        datasourceUid: prometheus
      nodeGraph:
        enabled: true

  - name: Pyroscope
    uid: pyroscope
    type: grafana-pyroscope-datasource
    access: proxy
    url: http://pyroscope:4040

deleteDatasources:
  - name: OldInflux
    orgId: 1
```

**Pinning `uid` is the whole game.** Dashboard JSON references datasources by UID, so a pinned UID makes every dashboard portable between Grafana instances. Without it, Grafana assigns a random UID per install and an imported dashboard shows “datasource not found”. Use short stable strings: `prometheus`, `loki`, `tempo`.

**`timeInterval` should match the Prometheus scrape interval.** It is what `$__rate_interval` is computed from, so getting it wrong reintroduces the empty-`rate()` problem that `$__rate_interval` exists to prevent.

The cross-signal configuration from [the Grafana page](../Observability/grafana.llms.md) lives here, which is the strongest practical argument for provisioning: the correlations are fiddly to set up by hand and trivial to copy as YAML.

### Dashboards

The provider tells Grafana where to find dashboard JSON.

``` yaml
apiVersion: 1

providers:
  - name: 'files'
    orgId: 1
    folder: 'Provisioned'
    type: file
    disableDeletion: true
    updateIntervalSeconds: 30
    allowUiUpdates: false           # true lets the UI save over them until restart
    options:
      path: /var/lib/grafana/dashboards
      foldersFromFilesStructure: true
```

`foldersFromFilesStructure: true` maps subdirectories to Grafana folders, which keeps a large set organised without listing every folder.

**Dashboard JSON needs two edits after export.** Strip the `id` field, which is instance-specific and causes collisions, and replace hard-coded datasource UIDs with a template variable if the dashboard should be portable across environments.

``` bash
# Export every dashboard from a running Grafana, cleaned
for uid in $(curl -sH "Authorization: Bearer $GRAFANA_TOKEN" \
               "$GRAFANA_URL/api/search?type=dash-db" | jq -r '.[].uid'); do
  curl -sH "Authorization: Bearer $GRAFANA_TOKEN" \
       "$GRAFANA_URL/api/dashboards/uid/$uid" \
  | jq '.dashboard | .id = null' > "dashboards/$uid.json"
done
```

### Alert Rules

Grafana’s unified alerting provisions the same way, which is what makes it comparable to file-based Prometheus rules.

``` yaml
apiVersion: 1

groups:
  - orgId: 1
    name: api
    folder: Alerts
    interval: 1m
    rules:
      - uid: api-error-ratio
        title: API error ratio above 5 percent
        condition: threshold
        data:
          - refId: query
            datasourceUid: prometheus
            model:
              expr: |
                sum(rate(http_requests_total{job="api",status=~"5.."}[5m]))
                  / sum(rate(http_requests_total{job="api"}[5m]))
              instant: true
          - refId: threshold
            datasourceUid: __expr__
            model:
              type: threshold
              expression: query
              conditions:
                - evaluator: {type: gt, params: [0.05]}
        for: 10m
        labels:
          severity: page
        annotations:
          summary: 'API error ratio is {{ $values.query }}'
        noDataState: NoData
        execErrState: Error
```

**`noDataState` and `execErrState` are the fields people forget.** `noDataState: NoData` fires a distinct no-data alert, which is usually right: a query returning nothing because the service vanished should not look like a healthy zero. Setting it to `OK` is how an alert silently stops protecting anything.

------------------------------------------------------------------------

## The Terraform Provider

Provisioning files are good for one Grafana. Terraform manages many, and manages the things files cannot: folders with permissions, service accounts, teams, organisations, and cloud stacks.

``` hcl
terraform {
  required_providers {
    grafana = {
      source  = "grafana/grafana"
      version = "~> 3.0"
    }
  }
}

provider "grafana" {
  url  = "http://192.168.2.205:3000"
  auth = var.grafana_service_account_token
}

resource "grafana_folder" "observability" {
  title = "Observability"
}

resource "grafana_data_source" "prometheus" {
  type       = "prometheus"
  name       = "Prometheus"
  uid        = "prometheus"
  url        = "http://prometheus:9090"
  is_default = true

  json_data_encoded = jsonencode({
    timeInterval = "15s"
    httpMethod   = "POST"
  })
}

resource "grafana_dashboard" "api" {
  folder      = grafana_folder.observability.uid
  config_json = file("${path.module}/dashboards/api-overview.json")
}

resource "grafana_rule_group" "api" {
  name             = "api"
  folder_uid       = grafana_folder.observability.uid
  interval_seconds = 60

  rule {
    name      = "API error ratio above 5 percent"
    condition = "threshold"
    for       = "10m"
    labels    = { severity = "page" }
    # data blocks as above
  }
}
```

This is the same Terraform covered in [the Terraform page](../terraform.llms.md), and the same workflow: `init`, `plan`, `apply`. The advantage over files is that `plan` shows exactly what will change before it changes, and state tracks what exists, so a deleted resource is actually removed rather than orphaned.

**Do not mix the two mechanisms on one object.** A dashboard provisioned from a file and also managed by Terraform will fight, with each apply and each Grafana restart overwriting the other. Pick one per object type. A common split is datasources and alerting by file, because they ship with the compose stack, and dashboards by Terraform, because they change more often and benefit from a plan.

**The token is a credential.** Use a service-account token with the minimum role, keep it in a variable, and keep it out of the repository. See the placeholder convention in [Grafana](../Observability/grafana.llms.md).

------------------------------------------------------------------------

## Grizzly

`grr` is a purpose-built CLI for Grafana objects, and it sits between hand-managed files and full Terraform.

``` bash
grr get Dashboard.api-overview > dashboards/api-overview.yaml   # pull one down
grr diff dashboards/                                            # what would change
grr apply dashboards/                                           # push it up
grr watch dashboards/ api-overview.yaml                         # live reload while editing
grr serve dashboards/                                           # preview without touching a server
```

`grr watch` is the feature that makes editing dashboard source bearable: save the file, see it in the browser immediately. It largely removes the argument that code-managed dashboards kill the feedback loop.

Grizzly also understands Jsonnet and Grafonnet, which is how large organisations generate hundreds of near-identical dashboards from a template rather than copying JSON. That is real power and a real learning curve; for a handful of dashboards, plain JSON plus `grr` is enough.

------------------------------------------------------------------------

## Choosing An Approach

| Situation | Use |
|----|----|
| One Grafana, ships with the compose stack | File provisioning |
| Several environments, want a plan before applying | Terraform provider |
| Editing dashboards often, want a fast loop | Grizzly, with files or Terraform underneath |
| Hundreds of generated dashboards | Jsonnet or Grafonnet via Grizzly |
| A dashboard somebody is still designing | The UI. Export and commit when it settles |

For a home lab, file provisioning committed next to the compose file covers everything, and the whole Grafana becomes reproducible from the repository.

------------------------------------------------------------------------

## What To Version

Everything that would have to be rebuilt after losing `grafana.db`:

- Datasources, with pinned UIDs and the cross-signal links.
- Dashboard JSON.
- Alert rules, contact points, notification policies, mute timings.
- Folder structure and permissions.
- Prometheus and Loki recording and alerting rules, which are already files.
- The compose file or Ansible role that ties it together.

What stays in the database and is therefore worth a backup anyway: users, API tokens, annotations, alert state history, and anything created ad hoc in the UI.

**The test is simple: delete the Grafana volume and bring it back up.** If the dashboards return and the alerts resume, the setup is genuinely reproducible. If it takes an afternoon of clicking, it was not.

------------------------------------------------------------------------

## Where Next

- [Grafana](../Observability/grafana.llms.md) for what is being provisioned.
- [Alerting](../Observability/alerting.llms.md) for the rules themselves.
- [Terraform](../terraform.llms.md) for the provider’s workflow.

------------------------------------------------------------------------

Back to top
