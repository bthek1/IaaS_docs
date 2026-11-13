# IaaS

Infrastructure as a Service, and the self-hosted stack that runs on it: Docker application containers, LXC/LXD system containers, and the services built on top.

Author

Benedict Thekkel

These are working notes on provisioning infrastructure and running services on it. The focus is practical. Every service page carries a compose file that has actually been run, not a paraphrase of the vendor documentation.

Most of it is exercised on a self-hosted Proxmox home lab rather than a public cloud, so the IaaS layer here is one you own and operate yourself.

------------------------------------------------------------------------

## The Service Model Stack

IaaS is the layer where you are handed compute, storage, and network, and everything above the hypervisor is yours to manage.

| Model | You manage | Provider manages | Examples |
|----|----|----|----|
| On-premises | Hardware, virtualization, OS, runtime, application, data | Nothing | A server you racked yourself |
| **IaaS** | OS, runtime, application, data | Hardware, virtualization, network | EC2, Linode, Proxmox VMs and containers |
| PaaS | Application, data | Everything below the runtime | Heroku, Fly.io, App Engine |
| SaaS | Data only | Everything | Gmail, Notion |

Self-hosting complicates that table in a useful way. Proxmox exposes the same abstraction a cloud IaaS does, an API that turns bare metal into VMs and containers on demand, except you are also the provider. You get the interface without outsourcing the hardware, and you inherit the operations work that a cloud provider would otherwise absorb.

------------------------------------------------------------------------

## Where Containers Fit

Containers are what you run on the IaaS layer, and the single word covers two quite different things. Getting the distinction right is usually what decides which tool a job wants.

|  | Virtual machine | System container (LXC/LXD) | Application container (Docker) |
|----|----|----|----|
| Holds | A full guest OS | The userspace of a full distro | One process and its dependencies |
| Kernel | Its own | Shared with the host | Shared with the host |
| Isolation | Hardware level, strongest | Namespaces and cgroups | Namespaces and cgroups |
| Starts in | Tens of seconds | Seconds | Milliseconds |
| Treat it as | A machine | A machine | A process |
| Reach for it when | You need a different OS or a hard isolation boundary | You want a long-lived host to SSH into and configure | You are packaging and shipping a single service |

[LXC and LXD](Docker/lxc_and_lxd.llms.md) covers the system container side. Everything else here is Docker.

------------------------------------------------------------------------

## Contents

**Provisioning the infrastructure**

| Page | Covers |
|----|----|
| [Terraform](terraform.llms.md) | Declarative IaC in HCL: providers, resources, state, and the init/plan/apply workflow, plus what the Proxmox provider does with it |
| [Ansible](ansible.llms.md) | Agentless configuration management over SSH: inventory, playbooks, roles, variables, idempotency, and where it beats or loses to Terraform |
| [Pulumi](pulumi.llms.md) | Terraform’s model driven from a real programming language, the `Output[T]` trap that catches everyone, and when the tradeoff is worth it |
| [K6](k6.llms.md) | Load testing with a Go binary running JS in an embedded runtime, which is where every gotcha comes from: script lifecycle, executors, checks versus thresholds, the URL cardinality trap in tagging, and distributed runs |

**Foundations**

| Page | Covers |
|----|----|
| [Docker Overview](Docker/docker_overview.llms.md) | Images, containers, and volumes, plus how a Dockerfile is layered instruction by instruction, with a CUDA PyTorch image as the worked example |
| [Install Docker](Docker/install_docker.llms.md) | Adding Docker’s apt repository on Linux, installing the engine, verifying it, and running it as a non-root user |
| [LXC and LXD](Docker/lxc_and_lxd.llms.md) | System containers: how they differ from Docker, `lxd init` explained option by option, then profiles, storage pools, networks, and the lifecycle commands |

**Orchestration**

| Page | Covers |
|----|----|
| [Kubernetes Overview](Kubernetes/kubernetes_overview.llms.md) | The reconcile loop, cluster components, the objects you actually write, probes and resource limits, the three networking layers, and when a single host means you should not use any of it |

**Services**

| Page | Covers |
|----|----|
| [PostgreSQL](Docker/postgresql.llms.md) | A `docker run` quick start through to a compose file with a persistent volume, plus connecting from another container, tuning, backups, and the usual traps |
| [Dockerized PostgreSQL](Docker/postgresql_docker.llms.md) | The same database wrapped in a Makefile for the day-to-day operations |
| [Nginx](Docker/nginx.llms.md) | Nginx on its own, then as a reverse proxy in front of an app container, then serving a static front end beside a proxied API |
| [JupyterLab](Docker/jupyterlabs.llms.md) | Compose file, volume permissions, and generating the password hash |
| [InfluxDB](Docker/influx_docker.llms.md) | Time series database whose entire initial setup is driven from environment variables |
| [Portainer](Docker/portainer.llms.md) | Web UI for managing the containers already running on a host |
| [Frigate](Docker/frigate.llms.md) | NVR with object detection, running inside a Proxmox LXC, including Coral TPU passthrough |
| [Kali Linux](Docker/kali.llms.md) | A full Kali desktop reached through the browser |
| [Firefox](Docker/firefox.llms.md) | Containerized browser served over VNC, mounted at a subfolder behind a reverse proxy |

**Observability: concepts and metrics**

| Page | Covers |
|----|----|
| [Observability Overview](Observability/observability_overview.llms.md) | What separates observability from monitoring, the four signals and what each costs, cardinality as the constraint behind every tool choice, pull versus push, and the four-layer model the rest of these pages sit in |
| [Prometheus](Observability/prometheus.llms.md) | The metrics database: data model and the four metric types, exposition format, scrape config, service discovery, both relabelling phases, TSDB sizing arithmetic, and the operational traps |
| [Exporters and Instrumentation](Observability/exporters_and_instrumentation.llms.md) | Where metrics come from: node_exporter, cAdvisor versus kube-state-metrics, the blackbox_exporter relabelling dance, why Pushgateway is usually wrong, and writing your own with `prometheus_client` |
| [PromQL](Observability/promql.llms.md) | Selectors, rate versus irate versus increase, counter resets, why aggregating a rate is not rating an aggregate, histogram quantiles, vector matching, recording rules, and a table of the traps |
| [Alerting](Observability/alerting.llms.md) | Alerting rules and what `for` really does, `promtool test rules`, the Alertmanager route tree and its four timers, grouping, inhibition, silences, and Grafana unified alerting compared |

**Observability: logs, traces and profiles**

| Page | Covers |
|----|----|
| [Loki](Observability/loki.llms.md) | The label-only index and when that bet fails, streams and chunks, the two config settings that silently destroy retention, the collection agents, and Loki versus Elasticsearch |
| [LogQL](Observability/logql.llms.md) | Stream selectors as the only indexed part, line filters before parsers, the five parsers, metric queries over log lines, alerting on a log line that never arrived, and a performance checklist |
| [Tempo](Observability/tempo.llms.md) | What a trace is and why broken ones are always a propagation bug, TraceQL including structural queries, head versus tail sampling, the metrics generator, and exemplars as the link from a graph to a real request |
| [Pyroscope](Observability/pyroscope.llms.md) | Continuous profiling: how to actually read a flame graph, CPU versus wall clock, push SDK versus scrape versus eBPF, and the four questions only a profiler answers |

**Observability: collection**

| Page | Covers |
|----|----|
| [OpenTelemetry](Observability/opentelemetry.llms.md) | The spec, SDKs and OTLP, automatic versus manual instrumentation, context propagation and the four places it breaks, semantic conventions, and the metric temporality trap |
| [OpenTelemetry Collector](Observability/otel_collector.llms.md) | Receivers, processors, exporters and connectors, why processor order is semantic, OTTL, the agent and gateway patterns, reliability, and sizing |
| [Alloy](Observability/alloy.llms.md) | Grafana’s Collector distribution: the component graph and the discovery reuse that justifies it, Alloy syntax, clustering, and when the difference from upstream actually matters |
| [Log Collectors](Observability/log_collectors.llms.md) | Fluent Bit and Vector as the non-Grafana alternatives, VRL and its test harness, and the six problems every log pipeline has regardless of agent |

**Observability: reading, operating, deciding**

| Page | Covers |
|----|----|
| [Grafana](Observability/grafana.llms.md) | Datasources and why UIDs must be pinned, Explore versus dashboards, the cross-signal links that make one click lead from a graph to a trace to a log, variables, transformations, and dashboard design |
| [Dashboards as Code](Observability/dashboards_as_code.llms.md) | File provisioning, the Terraform Grafana provider, Grizzly, what to version, and the delete-the-volume test for whether it is really reproducible |
| [Long-Term Storage](Observability/long_term_storage.llms.md) | Remote write and its cost controls, Mimir versus Thanos versus VictoriaMetrics on architecture rather than features, downsampling, why federation is not this, and a progression that starts by adding nothing |
| [SLOs and Alerting Practice](Observability/slos_and_alerting_practice.llms.md) | The four golden signals, RED and USE, error budgets with the downtime table, multi-window burn-rate alerting, what deserves a page, and how it all scales down to a home lab |
| [Observability Landscape](Observability/observability_landscape.llms.md) | The commercial platforms and their cost models, the ClickHouse-backed all-in-ones, the eBPF layer and its ceiling, the adjacent tooling, and where this stack’s tradeoff sits |
| [LGTM Stack](Observability/lgtm_stack.llms.md) | The whole thing in one compose file: OpenTelemetry Collector in front, Loki for logs, Tempo for traces, Mimir for metrics, Grafana on top, with the queries to read it back |

------------------------------------------------------------------------

## The Pattern Every Service Page Follows

The service pages are deliberately uniform: a `docker-compose.yml`, the command that brings it up, and the URL or port it lands on.

``` sh
docker compose up --build -d
```

Read the compose file before you run it. These examples carry placeholder values you are expected to change, including bind mounts pointing at `/path/to/data`, `PUID` and `PGID` fixed at 1000, and default credentials such as the InfluxDB admin password and the Firefox VNC password. They are fine on a private LAN and unsafe anywhere else.

Several pages also mount `/var/run/docker.sock` into the container. That grants the container root on the host, so treat it as a deliberate choice rather than boilerplate to copy.

------------------------------------------------------------------------

## Where This Runs

The host for most of these notes is a Proxmox home lab, which puts the IaaS layer, the containers, and the services on the same hardware.

- [Software Tools](https://bthek1.github.io/Software_Tools/) covers Proxmox itself, storage, and the surrounding Linux tooling.
- [Hardware Tools](https://bthek1.github.io/Hardware_Tools/) covers the machines underneath.
- [Back End](https://bthek1.github.io/Back_End/) covers the applications that get deployed into these containers.
- [The blog](https://bthek1.github.io/bthek1_blog/) has the narrative version of the same home lab build.

------------------------------------------------------------------------

## Not Covered Yet

Stated plainly, so the gaps are not mistaken for oversights:

- **Standing up a Kubernetes cluster.** The Kubernetes page covers the model and the manifests, not the install. No k3s or kubeadm walkthrough, and nothing on RBAC, storage classes, or autoscaling.

- **The lab’s own IaC.** The Terraform and Ansible pages are general. The code that actually provisions this home lab lives in the `infra/` directory of the parent Knowledge repo.

- **Public cloud IaaS.** No EC2, GCE, or equivalent.

- **Registries and CI.** Building, tagging, and pushing images to a registry.

- **Observability at real scale.** Every page in that section assumes one cluster or one machine. Nothing on running Mimir, Thanos, or a Loki cluster as a distributed system with the operational load that implies.

- **eBPF tooling in depth.** Beyla, Pixie, Coroot and Cilium are named and placed in the [landscape](Observability/observability_landscape.llms.md), not taught. No deployment or configuration for any of them.

- **Chaos engineering.** Litmus and Chaos Mesh get a mention only. Deliberately inducing failure is the honest test of whether the alerting works, and it is not covered here.

- **Cost observability.** OpenCost, Kubecost and Infracost are named, not used.

- **Real user monitoring and frontend performance.** Faro, the OTel browser SDK, Core Web Vitals and session replay. The observability pages stop at the server.

- **Security observability.** Falco, Wazuh, and the log-pipeline-to-detection path. Adjacent to the logging pages and a genuinely different subject.

The two PostgreSQL pages also overlap and are worth merging.

------------------------------------------------------------------------

Back to top
