# Kubernetes

Kubernetes is a control loop engine: you declare desired state as API objects, and controllers continuously reconcile actual state toward it. Everything else — pods, deployments, services — is just objects that particular controllers watch. Internalise that and the rest falls out.

Author

Benedict Thekkel

## The architecture

**Control plane:** - `etcd` — the only stateful component; the entire cluster is rows in this key-value store - `kube-apiserver` — the sole gateway to etcd; does authn, authz (RBAC), admission control, validation. Every other component talks only to this - `kube-scheduler` — assigns pending pods to nodes based on resource requests, affinity, taints - `kube-controller-manager` — the bundle of reconciliation loops (Deployment→ReplicaSet→Pod, node lifecycle, endpoints)

**Per node:** - `kubelet` — watches for pods assigned to its node, drives the container runtime (containerd), reports status - `kube-proxy` — programs iptables/nftables/IPVS rules so Service ClusterIPs resolve to pod IPs. Increasingly replaced by eBPF (Cilium) - CNI plugin — actual pod networking

------------------------------------------------------------------------

## Objects you actually use

| Object | Purpose |
|----|----|
| **Pod** | One or more co-located containers sharing a network namespace and volumes. You rarely create these directly |
| **Deployment** | Stateless workloads. Manages ReplicaSets; gives you rolling updates and rollback |
| **StatefulSet** | Stable identity + per-pod storage. Postgres, Redis. Usually a sign you should use managed services instead |
| **DaemonSet** | One pod per node. Log shippers, CNI, node exporters |
| **Job / CronJob** | Run-to-completion. Your Celery-style batch work, migrations |
| **Service** | Stable virtual IP + DNS name load-balancing to a label-selected set of pods. Types: ClusterIP, NodePort, LoadBalancer |
| **Ingress / Gateway API** | L7 HTTP routing, TLS termination |
| **ConfigMap / Secret** | Injected as env vars or mounted files. Secrets are base64, *not encrypted* by default — enable encryption at rest |
| **PVC / PV / StorageClass** | Storage claim, the actual volume, and the dynamic provisioner |
| **Namespace** | Soft isolation boundary. Not a security boundary on its own |
| **CRD + controller** | How you extend the API. This is the “operator pattern” |

**Labels and selectors are the connective tissue.** Services find pods by label, not by name. Deployments own ReplicaSets by label. Get sloppy with labels and you get very confusing bugs.

------------------------------------------------------------------------

## The parts that bite people

**Networking model.** Every pod gets a routable IP; all pods can reach all pods without NAT. The CNI plugin makes that true — Cilium (eBPF, best-in-class, includes network policy and can replace kube-proxy), Calico, or Flannel (simplest, least featureful). Default posture is *flat and open*: you need NetworkPolicy objects to restrict anything, and Flannel doesn’t implement them.

**Ingress is in flux.** Ingress NGINX — the de facto default for a decade — is being retired. There will be no further releases for bug fixes or security patches after retirement, and the recommended path is migration to Gateway API or third-party ingress controllers, none of which are drop-in replacements. If you’re starting now, learn **Gateway API** (`GatewayClass`/`Gateway`/`HTTPRoute`) rather than Ingress. Envoy Gateway or Cilium Gateway are reasonable implementations.

**Resource requests vs limits.** Requests drive scheduling; limits drive enforcement. CPU limits cause throttling (usually bad — consider omitting them). Memory limits cause OOMKill (necessary). Getting requests wrong is the \#1 cause of both bad density and mystery evictions. QoS class (Guaranteed/Burstable/BestEffort) determines eviction order.

**Probes.** `liveness` restarts the container, `readiness` pulls it from Service endpoints, `startup` covers slow boots. A liveness probe pointed at a dependency-checking endpoint will cascade-restart your whole cluster during a DB blip. Liveness should check *this process*, readiness can check dependencies.

**Storage is the hard part.** Stateful workloads want a CSI driver with real semantics — Longhorn or Rook/Ceph on-prem, cloud EBS/PD otherwise. `ReadWriteMany` is often what people want and rarely what they should build on.

------------------------------------------------------------------------

## Operating it

- **kubectl** — `get`, `describe`, `logs`, `exec`, `port-forward`, `apply`, and `events --sort-by`. `describe` and `events` solve ~70% of incidents.
- **Packaging** — Helm (templated YAML, de facto standard, painful string templating) or Kustomize (overlay-based patches, built into kubectl, no templating). For your own apps, Kustomize; for third-party charts, Helm.
- **GitOps** — Argo CD or Flux. Git is the source of truth, a controller reconciles the cluster to it. This is the correct way to run k8s and worth adopting from day one.
- **Observability** — Prometheus + Grafana (kube-prometheus-stack), Loki for logs, OpenTelemetry for traces.
- **Autoscaling** — HPA (pod count from metrics), VPA (right-sizes requests), Cluster Autoscaler / Karpenter (node count).
- **Security** — RBAC, Pod Security Admission (replaced PodSecurityPolicy), NetworkPolicy, external secrets (External Secrets Operator or Vault), image scanning, admission policies via Kyverno or the built-in ValidatingAdmissionPolicy (CEL).

------------------------------------------------------------------------

## Version reality

Current stable is the 1.36.x line, with 1.37 in beta. The project supports the three most recent minor versions on a roughly 15-week release cycle, giving each release about 14 months of life. That means **you will be upgrading two to three times a year, forever** — that’s the ops tax made concrete. Recent notables: in-place pod resource updates went stable, and user namespaces went GA, mapping container root to an unprivileged host user so a breakout grants no host admin rights.

------------------------------------------------------------------------

## Distributions

- **k3s** — single binary, low memory, batteries included. Right choice for your homelab. Run it in a few Proxmox VMs, or on Debian/Ubuntu guests.
- **Talos Linux** — immutable, API-driven OS with no shell. The most “correct” bare-metal option, excellent on Proxmox, steeper initial curve.
- **kubeadm** — the reference way to build a cluster manually. Do this once to understand the pieces.
- **kind / minikube** — local dev clusters.
- **EKS/GKE/AKS** — managed control plane. If you ever run this in production, use one of these; do not self-manage etcd.

------------------------------------------------------------------------

## Learning path I’d actually recommend

1.  `kind create cluster`, deploy a Django app + Postgres by hand-writing Deployment/Service/Ingress/ConfigMap/Secret YAML. No Helm. Feel the boilerplate.
2.  Break things deliberately: kill pods, set a bad liveness probe, exhaust memory limits, mislabel a selector. Debug with `describe`/`events`/`logs`.
3.  Build a 3-node k3s cluster on Proxmox VMs with Cilium and Longhorn. Add Argo CD pointed at a git repo.
4.  Read *Kubernetes Up & Running* alongside, and the API reference for the objects you touch.

Skip: writing your own operator, service mesh (Istio/Linkerd), and multi-cluster anything until you have a concrete problem that demands them.

------------------------------------------------------------------------

Back to top
