---
title: Observability Stack on Minikube — Setup & Learning Notes
tags: [observability, prometheus, loki, grafana, kubernetes, helm, sre-bootcamp]
date: 2026-08-10
status: in-progress
---

# Observability Stack on Minikube

Notes from building a Prometheus + Loki + Grafana stack on a 4-node minikube cluster.
Covers **what we installed, why each setting exists, and every problem we hit**.

---

## 1. The Big Picture

The most important idea: **there are two completely separate pipelines.**
Prometheus and Loki never talk to each other. They only meet inside Grafana.

```
METRICS  (Prometheus PULLS)              LOGS  (Loki RECEIVES pushes)

  node-exporter    ─┐                      pod writes to stdout
  kube-state-metrics┼──► Prometheus              │
  (later: postgres, │      scrapes         written to /var/log/pods on the node
   blackbox)       ─┘         │                  │
                              │            Promtail (DaemonSet) tails the file
                              │                  │ pushes
                              │                Loki
                              └────────┬─────────┘
                                       ▼
                                    Grafana
                            (2 datasources, queries both live)
```

| | Prometheus | Loki |
|---|---|---|
| Data | numbers | text lines |
| Direction | **pulls** from targets | **receives** pushes |
| Knows its sources? | yes, a target list | no, whoever POSTs |
| Query language | PromQL | LogQL |
| Indexes | everything | **labels only**, never log text |

> [!important] Grafana stores nothing
> No metrics, no logs. Every panel runs a live query. Delete Grafana and you lose zero data.

---

## 2. Core Concepts (in plain words)

### Exporter
A small program that translates something into text Prometheus can read over HTTP:

```
node_memory_MemAvailable_bytes 1.2345e+09
kube_pod_status_phase{pod="api-1",phase="Running"} 1
```

That's the whole concept. Nothing magic.

| Exporter | Reads from | Tells you | Runs as |
|---|---|---|---|
| **node-exporter** | `/proc`, `/sys` of the machine | CPU, RAM, disk, network | DaemonSet (1 per node) |
| **kube-state-metrics** | the Kubernetes API | replicas wanted vs ready, pod phases, restarts | Deployment (1 total) |

> [!tip] They cannot replace each other
> KSM = what Kubernetes **wants and thinks**. node-exporter = what the machine is **actually doing**.
> The API server does **not** store live CPU/RAM usage — that only exists in the kernel of each machine, right now. So you need a process *on* that machine.

### Who talks to the Kubernetes API?

A very common confusion. Answer:

| Component | Talks to K8s API? | Why |
|---|---|---|
| **Prometheus** | ✅ | to **find** what to scrape (discovery) |
| **kube-state-metrics** | ✅ | the API **is** its data |
| **node-exporter** | ❌ never | just reads local files |
| **Promtail** | ✅ | to **find** which log files to tail |

Proof: `node-exporter` has a ServiceAccount but **zero** ClusterRoleBindings. It isn't allowed to read a single Kubernetes object.

So there are two different uses of the API:
- **Discovery** — "where do I send requests?" (Prometheus's / Promtail's problem)
- **Data source** — "what facts do I publish?" (each exporter's own problem)

### Service Discovery
You never write a list of IP addresses. You tell Prometheus to ask Kubernetes:

```yaml
kubernetes_sd_configs:
  - role: endpoints        # give me pod addresses behind Services
    namespaces:
      names: [observability-ns]
```

| role | gives you |
|---|---|
| `endpoints` | individual pod addresses behind each Service ← most used |
| `pod` | every pod |
| `node` | every node |
| `service` | Service names (used for blackbox probing) |

> [!warning] Why `endpoints` and not the Service IP
> A Service ClusterIP is a **load balancer**. Scraping it would hit a *random* node-exporter each time and store all 4 nodes' data under one shifting identity — silently wrong.
> `role: endpoints` bypasses it: each pod address becomes its own target.
> The Service is used as a **registry**, not as a load balancer.

### Relabeling
Discovery returns **everything**. Relabeling is a filter that runs **before** scraping.

Each candidate arrives with hidden labels named `__meta_kubernetes_*`.
Labels starting with `__` are internal and thrown away before storage.

Three special ones:
- `__address__` — where to scrape *(Prometheus)*
- `__path__` — which file to tail *(Promtail)*
- `__metrics_path__` — defaults to `/metrics`

| action | meaning |
|---|---|
| `keep` | throw away everything that does NOT match |
| `drop` | throw away everything that DOES match |
| `replace` | copy a value into a label |
| `labelmap` | bulk-copy labels matching a pattern |

Also: `relabel_configs` filters **targets** (before scraping); `metric_relabel_configs` filters **samples** (after scraping).

> [!tip] Relabeling is a shared language
> Prometheus and Promtail use the **exact same syntax**. Learn it once, use it twice.
> This is also *why* correlation works in Grafana — both pipelines label data the same way (`namespace`, `pod`, `container`).

### Metric types
| type | meaning | how to query |
|---|---|---|
| counter | only goes up (e.g. seconds since boot) | **always** wrap in `rate()` |
| gauge | goes up and down (e.g. free memory) | read directly |
| histogram | buckets, for latency percentiles | `histogram_quantile()` |

> [!bug] The #1 dashboard mistake
> Graphing a counter raw gives a meaningless ever-rising line. There is no `node_cpu_usage_percent` — Prometheus exports raw facts and expects you to do the math at query time.

### Loki: label cardinality
Loki indexes **only labels**. A query runs in two phases:

```
{namespace="student-api"} |= "error"
└─── label selector ────┘ └─ filter ─┘
  uses the index           greps the matching chunks
```

> [!danger] Never put high-cardinality values in a Loki label
> `request_id`, `user_id`, `trace_id` → each unique value creates a new **stream** with its own chunks. Millions of streams = Loki falls over, unrecoverably.
> Keep variable data **inside the log line** and filter with `|=` or `| json` at query time.

---

## 3. Cluster Setup

Our 4-node minikube:

| Node | Label | Purpose |
|---|---|---|
| `minikube` | control-plane | cordoned (`SchedulingDisabled`) |
| `minikube-m02` | `type=database` | Postgres |
| `minikube-m03` | `type=dependent_services` | **observability stack** |
| `minikube-m04` | `type=application` | student-api |

### Prerequisites

```bash
# 1. Namespace
kubectl create namespace observability-ns

# 2. Helm repos
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# 3. Storage provisioner (REQUIRED on multi-node minikube)
minikube addons enable storage-provisioner-rancher
kubectl get sc     # expect: local-path
```

> [!warning] Why `local-path` and not the default `standard`
> | | `standard` | `local-path` |
> |---|---|---|
> | Binding mode | `Immediate` | `WaitForFirstConsumer` |
> | Multi-node safe | ❌ | ✅ |
>
> `Immediate` binds the volume **before** the scheduler picks a node. The folder gets created on the *provisioner's* node, but the pod runs elsewhere — so it mounts an empty root-owned folder it can't write to and crash-loops.
> `WaitForFirstConsumer` waits for scheduling first, then creates the folder on the **right** node.
>
> `fsGroup` does NOT fix this. Kubelet won't chown hostPath volumes. **It is a storage-class problem, not a securityContext problem.**

### The habit that saves hours

**Never install blind.** Render locally first — no cluster involved:

```bash
helm template <release> <chart> -n observability-ns -f <values.yml> | less
helm show values <chart> > /tmp/defaults.yml     # the REAL documentation
```

> [!danger] Helm does not error on unknown keys
> A typo like `nodeSelctor` is silently ignored, forever. The render is the truth; your values file is only half of it.

---

## 4. Installation, Step by Step

### Step 1 — kube-state-metrics

**What it is:** a Kubernetes API client that publishes object state as metrics.
**Why a Deployment, not a DaemonSet:** there's only one API server view. Running it per-node would create N identical copies of every metric.
**RBAC:** needs a cluster-wide **read-only** ClusterRole (`get/list/watch` on everything). Read verbs only is what makes such broad access safe.

📄 `observability/helm-values/kube-state-metrics-values.yml`

```yaml
# Pin to the dependent_services node, per the milestone's deployment diagram.
nodeSelector:
  type: dependent_services

# Single replica: KSM mirrors one API server view.
replicas: 1

service:
  # ClusterIP is right: only Prometheus (in-cluster) consumes this.
  # Never expose an exporter outside the cluster.
  type: ClusterIP
  port: 8080
```

```bash
helm upgrade --install kube-state-metrics prometheus-community/kube-state-metrics \
  -n observability-ns -f observability/helm-values/kube-state-metrics-values.yml
```

**Verify — look at the raw metrics with your own eyes:**
```bash
kubectl port-forward -n observability-ns svc/kube-state-metrics 8080:8080
curl -s localhost:8080/metrics | grep kube_deployment_status_replicas_available
```
This is *exactly* what Prometheus will see. No agent, no push, no magic — just text over HTTP.

> [!note] Service name
> Release name `kube-state-metrics` + chart name `kube-state-metrics` → Service is just `kube-state-metrics`. Write it down; Step 3 needs to match it exactly.

---

### Step 2 — node-exporter

**What it is:** reads the kernel's `/proc` and `/sys` and republishes them.
**Why a DaemonSet:** `/proc` is per-machine. To read node A's memory, you must run on node A. There's no remote API for it.

| | kube-state-metrics | node-exporter |
|---|---|---|
| Kind | Deployment, 1 replica | DaemonSet, 1 per node |
| Data source | API server (one, remote) | `/proc` (local, per-node) |
| `nodeSelector` | `type: dependent_services` | **none — must be everywhere** |

**Why it needs host access:** the rendered DaemonSet mounts `/proc`, `/sys`, `/` as hostPath, plus `hostPID: true` and `hostNetwork: true`. A container is normally isolated from exactly these. node-exporter's whole job is to *defeat* that isolation for read-only observation.

📄 `observability/helm-values/node-exporter-values.yml`

```yaml
# DaemonSet: intentionally NO nodeSelector.
# node-exporter reads /proc and /sys, which are per-machine -- it must run
# on every node or that node has no metrics at all.
nodeSelector: {}

# Tolerate everything. A node that is tainted, cordoned, or draining is
# exactly the node you most want metrics from.
tolerations:
  - operator: Exists

service:
  type: ClusterIP
  port: 9100

# The chart's fullname is <release>-<chart>, giving an awkward
# "node-exporter-prometheus-node-exporter". Pin it so the Service name is
# predictable -- Step 3's relabel rule has to match this string exactly.
fullnameOverride: node-exporter
```

```bash
helm upgrade --install node-exporter prometheus-community/prometheus-node-exporter \
  -n observability-ns -f observability/helm-values/node-exporter-values.yml
```

> [!tip] Surprise: 4 pods, not 3
> The control-plane node is **cordoned**, so a normal Deployment can't schedule there. But we got 4 node-exporters — because the **DaemonSet controller automatically injects a toleration** for `node.kubernetes.io/unschedulable`. DaemonSets land on cordoned nodes that reject every other workload.

**Verify:**
```bash
kubectl port-forward -n observability-ns svc/node-exporter 9100:9100
curl -s localhost:9100/metrics | grep '^node_cpu_seconds_total' | head

kubectl get endpoints node-exporter -n observability-ns
# → one Service, 4 addresses. THIS is what `role: endpoints` discovers.
```

Note the addresses are `192.168.49.x` — **node** IPs, not pod IPs. That's `hostNetwork: true` showing itself.

---

### Step 3 — Prometheus server

**What Prometheus is:** four things in one binary — scraper, TSDB (local disk), query engine, rule evaluator. No agents, no push endpoint, no clustering, no long-term storage.

**What the chart deploys:**
- Deployment with 2 containers: `prometheus-server` + a `configmap-reload` sidecar that POSTs to `/-/reload` when config changes
- ConfigMap holding `prometheus.yml`, mounted at `/etc/config/prometheus.yml`
- PVC for the TSDB
- ClusterRole (read-only) so `kubernetes_sd_configs` can query the API

📄 `observability/helm-values/prometheus-values.yml`

```yaml
# Turn off the bundled sub-charts (we install those ourselves).
# Careful: these names must match the sub-chart names exactly. Helm ignores
# keys it doesn't recognise WITHOUT an error.
alertmanager:
  enabled: false
kube-state-metrics:
  enabled: false
prometheus-node-exporter:
  enabled: false
prometheus-pushgateway:
  enabled: false

# Delete the chart's 10 built-in scrape jobs. See "Problem 2" below for why
# `null` and not `{}`.
scrapeConfigs: null

server:
  replicaCount: 1

  nodeSelector:
    type: dependent_services

  # Kill the old pod BEFORE starting the new one. The default RollingUpdate
  # starts the new pod first, but both want the same disk and only one can
  # have it -- so the new pod hangs in Pending forever.
  strategy:
    type: Recreate

  persistentVolume:
    enabled: true
    size: 10Gi
    storageClass: local-path     # multi-node safe (see Prerequisites)

  retention: 15d                 # recent operational data, not an archive

  service:
    type: ClusterIP

  global:
    scrape_interval: 30s         # how often to poll each target
    scrape_timeout: 10s          # must be < scrape_interval
    evaluation_interval: 30s

  securityContext:
    runAsUser: 65534             # "nobody"
    runAsGroup: 65534
    fsGroup: 65534

# This IS the prometheus.yml file, rendered into a ConfigMap.
serverFiles:
  prometheus.yml:
    scrape_configs:

      # ---- 1. Prometheus watching itself -------------------------------
      # If monitoring breaks, you need metrics about the monitoring.
      - job_name: prometheus
        static_configs:
          - targets:
              - localhost:9090

      # ---- 2. node-exporter --------------------------------------------
      - job_name: node-exporter
        kubernetes_sd_configs:
          - role: endpoints
            namespaces:
              names: [observability-ns]     # don't search the whole cluster

        relabel_configs:
          # Keep only node-exporter's endpoints; drop everything else.
          # WARNING: wrong name here = 0 targets, with NO error anywhere.
          - source_labels: [__meta_kubernetes_service_name]
            regex: node-exporter
            action: keep

          # Name each target after its node, so graphs read "minikube-m02"
          # instead of "192.168.49.3:9100" (and IPs change).
          - source_labels: [__meta_kubernetes_endpoint_node_name]
            target_label: instance
            action: replace

      # ---- 3. kube-state-metrics ---------------------------------------
      - job_name: kube-state-metrics
        kubernetes_sd_configs:
          - role: endpoints
            namespaces:
              names: [observability-ns]

        relabel_configs:
          - source_labels: [__meta_kubernetes_service_name]
            regex: kube-state-metrics
            action: keep

        # No `instance` rewrite here, unlike node-exporter.
        # node-exporter's numbers describe the node it runs on, so the node
        # name is real information. KSM reports on the WHOLE cluster and just
        # happens to be scheduled somewhere -- labelling it with a node name
        # would suggest the data belongs to that node.
```

> [!tip] The general rule
> **A label should describe the data, not the accident of where the process runs.**

```bash
helm upgrade --install prometheus prometheus-community/prometheus \
  -n observability-ns -f observability/helm-values/prometheus-values.yml

kubectl port-forward -n observability-ns svc/prometheus-server 9090:80
```

**Verify — the Targets page** (`Status → Target health`) is the best debugging tool in Prometheus:

| What you see | Meaning |
|---|---|
| Job missing / **0 targets** | your `keep` matched nothing — regex or namespace is wrong |
| Target listed, **DOWN** + error | discovery worked, scraping failed — wrong port/path/network |
| **UP** | working |

> [!warning] Those first two look identical from a dashboard ("no data") and have opposite fixes. Telling them apart is the skill.

Also check `Status → Configuration` — the live config as Prometheus parsed it. If it doesn't match your values file, the problem is Helm, not Prometheus.

**First queries:**
```promql
up                                                              # 1 or 0 per target
rate(node_cpu_seconds_total{mode="idle"}[5m])                   # idle fraction per core
100 * (1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])))   # CPU% per node
kube_deployment_spec_replicas - kube_deployment_status_replicas_available       # rollout gap
```

---

### Step 4 — Loki

**Deployment modes:** SingleBinary (1 pod) / SimpleScalable (9 pods, the **chart default**) / Distributed (15+). Same binary; the mode just decides which components each process runs.

> [!danger] This chart's defaults will not work on a laptop
> | Default | Problem |
> |---|---|
> | `deploymentMode: SimpleScalable` | 9 pods |
> | `chunksCache` `allocatedMemory: 8192` | memcached asking **8 GB** — Pending forever |
> | `resultsCache` | another 1 GB of memcached |
> | `storage.type: s3` (endpoint `null`) | assumes AWS, crashes |
> | `schemaConfig: {}` | empty — Loki won't start |
> | `auth_enabled: true` | needs `X-Scope-OrgID` header or 401 |
> | `replication_factor: 3` | needs 3 ingesters |
> | `singleBinary.replicas: 0` | no Loki at all |

📄 `observability/helm-values/loki-values.yml`

```yaml
# Run everything in one pod (default is a 9-pod read/write/backend split).
deploymentMode: SingleBinary

loki:
  # Means "no MULTI-TENANCY", not "no security". If left true, every request
  # needs an X-Scope-OrgID header or gets a 401.
  auth_enabled: false

  commonConfig:
    replication_factor: 1   # default 3 wants 3 ingesters; we have 1

  storage:
    type: filesystem        # default is s3 with empty credentials

  # Index layout on disk. Chart ships {} and Loki won't start without it.
  # `from` is a schema boundary -- to change format later, APPEND a new entry
  # with a future date. Never edit this one or old data becomes unreadable.
  schemaConfig:
    configs:
      - from: "2024-04-01"
        store: tsdb
        object_store: filesystem
        schema: v13
        index:
          prefix: index_
          period: 24h

singleBinary:
  replicas: 1               # default is 0 -- without this, no Loki
  nodeSelector:
    type: dependent_services
  persistence:
    enabled: true
    size: 10Gi
    storageClass: local-path

# Required: these default to 3 each, and the chart REFUSES TO RENDER when
# both singleBinary and these are non-zero.
write:
  replicas: 0
read:
  replicas: 0
backend:
  replicas: 0

gateway:          # nginx proxy for the read/write split -- pointless with 1 pod
  enabled: false
chunksCache:      # memcached, wants 8Gi
  enabled: false
resultsCache:
  enabled: false
lokiCanary:       # canary DaemonSet + helm-test pod: prod-useful, noise here
  enabled: false
test:
  enabled: false
```

```bash
# Sanity check FIRST -- if you see a Deployment named loki-chunks-cache or
# anything -write/-read/-backend, a disable didn't take.
helm template loki grafana/loki -n observability-ns \
  -f observability/helm-values/loki-values.yml | grep -E "^kind:"

helm upgrade --install loki grafana/loki \
  -n observability-ns -f observability/helm-values/loki-values.yml
```

**Verify:**
```bash
kubectl port-forward -n observability-ns svc/loki 3100:3100
curl -s localhost:3100/ready                        # → ready
curl -s localhost:3100/loki/api/v1/labels           # → empty, and that's CORRECT
```
Empty is right — nothing is sending yet. Loki is an empty inbox.

> [!note] Why this values file is so long
> How lean a values file can be is set by the **chart's defaults**, not your needs.
> Prometheus's file is short because that chart's defaults are close to what we wanted. Loki's is long because the chart assumes AWS + 9 pods, so most of the file is *declining* things.
> A long values file usually means "this chart was built for a different environment than mine" — useful signal, not your mistake.

---

### Step 5 — Promtail

**What it is:** a DaemonSet that tails log files on each node and POSTs them to Loki.

```
container writes to stdout
        ↓ (containerd captures it)
/var/log/pods/<ns>_<pod>_<uid>/<container>/0.log
        ↓ (Promtail tails)
     Promtail ──HTTP POST──► Loki
```

Two mounts make it work: `/var/log/pods` (the files) and a `positions.yaml` (how far it has read), so a restart resumes instead of re-sending everything.

**The link back to Step 3** — Promtail uses the *same* discovery + relabeling:

| | Prometheus | Promtail |
|---|---|---|
| role | `endpoints` | `pod` |
| Asking | "what addresses do I scrape?" | "what log files do I tail?" |
| Magic label | `__address__` | **`__path__`** |

**`pipeline_stages: [cri: {}]`** — Kubernetes doesn't store raw log lines. The runtime wraps each one:
```
2026-08-10T09:14:22.13Z stdout F {"level":"info","msg":"request handled"}
└── timestamp ─────────┘ └str┘ │ └── your actual line ─────────────────┘
```
The `cri` stage unwraps it: real timestamp, drops the wrapper. It's a chart default — you get it for free.

📄 `observability/helm-values/promtail-values.yml`

```yaml
config:
  clients:
    # Chart default points at loki-gateway, which we DISABLED in Step 4 --
    # so this MUST be overridden or Promtail fails DNS forever.
    - url: http://loki.observability-ns.svc.cluster.local:3100/loki/api/v1/push

  snippets:
    # Appended to the chart's own relabel rules.
    # Discovery finds EVERY pod in the cluster; the milestone wants only
    # application logs. Dropping here means those lines are never read or
    # sent: less traffic, less storage, fewer Loki streams.
    extraRelabelConfigs:
      - source_labels: [__meta_kubernetes_namespace]
        regex: student-api
        action: keep

# Our keep-filter means Promtail only has work on nodes running student-api.
# Idle instances report themselves unready, which blocks the default
# one-at-a-time DaemonSet rollout indefinitely. Replace all pods at once.
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 100%
```

```bash
helm upgrade --install promtail grafana/promtail \
  -n observability-ns -f observability/helm-values/promtail-values.yml
```

**Verify:**
```bash
# What Promtail actually loaded
kubectl get secret promtail -n observability-ns \
  -o jsonpath='{.data.promtail\.yaml}' | base64 -d

# Which files are being tailed
kubectl logs -n observability-ns -l app.kubernetes.io/name=promtail --tail=300 \
  | grep "tail routine: started"

# THE milestone check -- must be student-api only
kubectl exec -n observability-ns deploy/grafana -- wget -qO- \
  'http://loki.observability-ns.svc.cluster.local:3100/loki/api/v1/label/namespace/values'
```

Final result:
```json
{"status":"success","data":["student-api"]}
```

---

### Step 6 — Grafana

**Datasource provisioning = config as code.** You *could* click through the UI, but that state lives only on one pod's disk, invisible to git, gone on reinstall.

**`access: proxy`** — Grafana's *backend* makes the query, so URLs are in-cluster DNS names. (`direct` would mean the *browser* queries, which can't reach a ClusterIP.) This is why Prometheus and Loki never need to be exposed: **Grafana is the single front door.**

📄 `observability/helm-values/grafana-values.yml`

```yaml
nodeSelector:
  type: dependent_services

service:
  type: ClusterIP

persistence:
  enabled: true        # chart default is false
  size: 5Gi
  storageClassName: local-path

datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:

      # --- Metrics ---
      # NOTE: port 80, not 9090! The Service listens on 80 and forwards to
      # the container's 9090. 9090 is only what we port-forward locally.
      - name: Prometheus
        type: prometheus
        uid: prometheus
        url: http://prometheus-server.observability-ns.svc.cluster.local
        access: proxy
        isDefault: true

      # --- Logs ---
      - name: Loki
        type: loki
        uid: loki
        url: http://loki.observability-ns.svc.cluster.local:3100
        access: proxy

# No adminPassword here on purpose -- it would be a plaintext credential in
# git. Left unset, the chart generates a random one into a Secret.
# Later: wire to Vault via ESO with  admin: { existingSecret: grafana-admin }
```

> [!warning] `prometheus-server` listens on port **80**, not 9090
> The most common Grafana datasource failure. 9090 is only the local port-forward.

```bash
helm upgrade --install grafana grafana/grafana \
  -n observability-ns -f observability/helm-values/grafana-values.yml

kubectl get secret grafana -n observability-ns \
  -o jsonpath='{.data.admin-password}' | base64 -d; echo

kubectl port-forward -n observability-ns svc/grafana 3000:80
```

Login as `admin` → **Connections → Data sources** → *Save & test* on both. Green or fix the URL.

> [!note] `uid:` matters
> Dashboards reference datasources by uid. Setting them explicitly means a committed dashboard JSON resolves correctly on a fresh install instead of pointing at a random generated id.

**The payoff — correlation:**
1. Explore → Prometheus: `rate(node_cpu_seconds_total{mode="idle"}[5m])`, find a dip
2. Explore → Loki, same time range: `{namespace="student-api"}`
3. Use **Split** to view both side by side

*"CPU spiked at 14:32 — what was the app logging at 14:32?"*
Metrics tell you **something is wrong**; logs tell you **what**.

> [!tip] Why correlation works at all
> Both pipelines label data the same way (`namespace`, `pod`, `container`) because both got those labels from `kubernetes_sd_configs`. **The shared relabeling vocabulary is what makes it possible — not a Grafana feature.**

---

## 5. Problems We Hit

### Problem 1 — Prometheus CrashLoopBackOff (permission denied on `/data`)

| | |
|---|---|
| **Symptom** | Pod crash-loops; storage errors on `/data` |
| **Cause** | Default `standard` StorageClass is `Immediate` binding + hostPath. Volume bound before scheduling → folder created on the wrong node → pod mounts an empty root-owned dir |
| **Why `fsGroup` didn't help** | Kubelet only applies fsGroup to volume types it manages. hostPath isn't one — it refuses to chown arbitrary host dirs |
| **Fix** | `minikube addons enable storage-provisioner-rancher`, then `storageClass: local-path` (WaitForFirstConsumer) |

> [!important] It was a storage-class problem, not a securityContext problem.

---

### Problem 2 — `found multiple scrape configs with job name "prometheus"`

| | |
|---|---|
| **Symptom** | Prometheus exits immediately; config parse error |
| **Cause** | The chart builds `scrape_configs` from **two** places and concatenates them |

```
templates/cm.yaml:
    scrape_configs:
{{- range $root.Values.scrapeConfigs }}      ← chart's 10 defaults (a MAP)
{{- toYaml $value.scrape_configs }}          ← ours (a LIST), appended after
```

The defaults live at top-level `scrapeConfigs`, a **map keyed by job name** — a *different key* from the list we wrote. So our list replaced nothing, and `prometheus` was defined twice.

**Fix:** `scrapeConfigs: null`

> [!important] Helm merge rules differ by type — so check the **type** of the key you're overriding
> | Type | Behaviour | To remove |
> |---|---|---|
> | list | replaced wholesale | `[]` |
> | map | merged key-by-key | **`null`** (`{}` does nothing) |
>
> You cannot clear a map with `{}` — an empty map merges into the defaults and changes nothing.

**How we found it in one command:**
```bash
helm template prometheus prometheus-community/prometheus \
  -n observability-ns -f observability/helm-values/prometheus-values.yml | grep job_name
```

---

### Problem 3 — Promtail using 100% chart defaults

| | |
|---|---|
| **Symptom** | Rendered config showed `url: http://loki-gateway/...` and no `keep` rule |
| **Cause** | The `helm install` ran **without `-f`**. Helm never saw the file |
| **Fix** | Re-run with `-f`; confirm with `helm get values` |

Also found: stray backticks from a copy-paste (`action: keep\`\``) — valid YAML, but Promtail would reject it.

> [!important] The debugging ladder
> Three layers between your file and the running pod. When a setting doesn't take effect, walk them **in order**:
>
> | Layer | Check | Answers |
> |---|---|---|
> | 1. Your file | `cat` it | Is the YAML what I think? |
> | 2. What Helm received | `helm get values <rel> -n <ns>` | Did my `-f` land? |
> | 3. What Helm rendered | `helm get manifest` / the ConfigMap | Did my **key** take effect? |
>
> Same symptom ("my setting is ignored"), opposite fixes:
> - **Layer 2 fails** → missing `-f`, wrong path, wrong release name
> - **Layer 3 fails** → correct file, wrong *key* (Problem 2)
>
> `USER-SUPPLIED VALUES: null` isolates it instantly.

---

### Problem 4 — Config was correct, but no logs reached Loki

The best one. A three-link chain where **nothing was misconfigured**.

```
REVISION 1  17:54  install (no -f)     → all 4 pods created   (120m old)
REVISION 2  17:57  upgrade (with -f)   → ONE pod replaced     (117m old, 0/1)
```

1. **The DaemonSet rollout stalled.** DaemonSets roll one pod at a time (`maxUnavailable: 1`) and wait for each replacement to be **Ready**.
2. **Why that pod couldn't become Ready — our own filter.** Promtail reports itself unready with **zero active targets**. Our `keep` limits it to namespace `student-api`, which only runs on m02 and m04. The pod on m03 had nothing to tail → `/ready` returns 500 → never Ready → rollout blocked forever.
3. **So the other 3 pods still ran the OLD config**, pushing to the non-existent `loki-gateway`. Nothing reached Loki → no labels → empty Grafana label browser.

**The subtle bit:** we checked the config and it looked right —

```bash
$ kubectl exec promtail-w56jt -- cat /etc/promtail/promtail.yaml
clients:
  - url: http://loki.observability-ns.svc...   ← correct!
```

> [!danger] A mounted config file is NOT the running config
> Kubelet updates mounted Secret/ConfigMap contents **in place**, but most processes parse config once at startup and never re-read it. Promtail has no reloader.
> Prometheus gets away with this only because it ships a `configmap-reload` sidecar.
> **Check pod age against your last upgrade before you trust `cat`.**

**Fix:**
```bash
kubectl delete pod -n observability-ns -l app.kubernetes.io/name=promtail   # immediate
```
```yaml
updateStrategy:            # durable
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 100%
```

> [!important] Two more lessons
> - **Readiness probes gate rollouts, not just traffic.** A permanently-unready pod silently freezes a rollout. `helm upgrade` reports success either way — Helm doesn't wait unless you pass `--wait`.
> - **Our filtering decision had a second-order effect.** Narrow filter → idle agents → unready → frozen rollout. Nothing was wrong; the pieces just interacted. That's the normal texture of distributed systems.

---

### Problem 5 — Wrong Service names in relabel rules

| | |
|---|---|
| **Symptom** | Job shows **0 targets**. No error, no log line, nothing |
| **Cause** | `keep` regex didn't match the Helm-generated Service name (`node-exporter-prometheus-node-exporter`) |
| **Fix** | `fullnameOverride: node-exporter` — declare the name instead of discovering it |

> [!warning] Silent failure by design
> `keep` dropped every target. Prometheus did exactly what it was told. Always cross-check `kubectl get svc` against your regex.

---

## 6. Command Cheat Sheet

```bash
# --- Before installing anything ---
helm show values <chart> > /tmp/defaults.yml          # the real docs
helm template <rel> <chart> -n <ns> -f <values> | less # render locally

# --- The debugging ladder ---
cat <values.yml>                                       # layer 1
helm get values <rel> -n <ns>                          # layer 2  (null = no -f!)
helm get manifest <rel> -n <ns>                        # layer 3

# --- Did the pod actually restart? ---
kubectl get pods -n observability-ns -o wide           # compare AGE to upgrade time
helm history <rel> -n <ns>

# --- Why is it broken? ---
kubectl describe pod <pod> -n <ns>                     # scheduling / probes / mounts
kubectl logs <pod> -n <ns> -c <container> [--previous] # why the PROCESS exited
kubectl get events -n <ns>                             # what the KUBELET did

# --- UIs ---
kubectl port-forward -n observability-ns svc/prometheus-server 9090:80
kubectl port-forward -n observability-ns svc/grafana 3000:80
kubectl port-forward -n observability-ns svc/loki 3100:3100

# --- Loki without port-forward ---
kubectl exec -n observability-ns deploy/grafana -- wget -qO- \
  'http://loki.observability-ns.svc.cluster.local:3100/loki/api/v1/labels'
```

> [!tip] Events vs Logs
> | | Tells you |
> |---|---|
> | **Events** | what the **kubelet** did — scheduled, pulled, mounted, restarted |
> | **Logs** | what the **process** did, and why it exited |
>
> A config parse error will **never** appear in events. Kubelet only saw a non-zero exit code.

---

## 7. Progress

| Milestone expectation | Status |
|---|---|
| Prometheus, Loki, Grafana on `dependent_services` in observability ns | ✅ |
| Promtail sends **only** application logs | ✅ |
| Prometheus scrapes kube-state + node metrics | ✅ |
| Grafana has Loki + Prometheus datasources | ✅ |
| DB metrics exporter (postgres-exporter) | ⬜ |
| Blackbox exporter for internal endpoints | ⬜ |
| Charts/configs committed at proper path | ⬜ |
| README updated | ⬜ |

### Still to do
- **postgres-exporter** — first component needing credentials → plugs into Vault/ESO
- **blackbox-exporter** — different pattern: Prometheus scrapes *blackbox*, passing the target URL as a `?target=` param via relabeling
- **Argo CD wiring** — third-party chart + values from git
- **README** — setup instructions

### Open decisions worth writing down
- **Filter at Promtail vs at query time.** We drop non-app logs at the agent: cheap, but we lose visibility when the app breaks because of something in `kube-system`. Every logging pipeline makes this tradeoff.
- **Init container logs.** `run-migrations` is in the `student-api` namespace, so it ships too. `container` is already a label, so it can be filtered at query time instead.
- **Promtail is end-of-life.** Deprecated upstream in favour of **Grafana Alloy**. Used here because the bootcamp specifies PLG.

---

## 8. Reference Links

**Concepts**
- [Kubernetes monitoring with Prometheus, the ultimate guide (Sysdig)](https://www.sysdig.com/blog/kubernetes-monitoring-prometheus) — best free explanation of discovery + relabeling
- [kube-state-metrics: A Guide (Last9)](https://last9.io/blog/kube-state-metrics/)
- [Setup Prometheus Node Exporter on Kubernetes (DevOpsCube)](https://devopscube.com/node-exporter-kubernetes/) — raw manifests, nothing hidden

**Helm specifics**
- [Configuring Prometheus with Helm Chart (Spacelift)](https://spacelift.io/blog/prometheus-helm-chart)
- [Prometheus Monitoring for Kubernetes Cluster (Spacelift)](https://spacelift.io/blog/prometheus-kubernetes)

**Loki / PLG**
- [Setup Grafana Loki on Kubernetes & Query Logs (DevOpsCube)](https://devopscube.com/setup-grafana-loki/)
- [Logging in Kubernetes with Loki and the PLG stack (Thorsten Hans)](https://www.thorsten-hans.com/logging-in-kubernetes-with-loki-and-plg-stack/) — why Loki differs from ELK
- [Deploying PLG stack on kubernetes (Antti Viitala)](https://aviitala.com/posts/deploy-plg-stack/)

**Official docs**
- [Prometheus configuration reference](https://prometheus.io/docs/prometheus/latest/configuration/configuration/) — every `relabel_configs` field
- [Loki deployment modes](https://grafana.com/docs/loki/latest/get-started/deployment-modes/)
