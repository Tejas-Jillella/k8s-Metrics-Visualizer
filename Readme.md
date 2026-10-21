# k8s Metrics Visualizer

Real-time CPU, memory and GPU observability for a 9-node, NVIDIA L40S Kubernetes cluster, built on
Prometheus and Grafana. Everything is declarative: four manifests deploy the full stack, including
the dashboards.

## What it does

- Scrapes GPU, node and container metrics every 5 seconds and keeps 30 days of history.
- Provides four Grafana dashboards in three folders, from a quick cluster snapshot down to
  per-core CPU and per-GPU detail on a single node.
- Provisions dashboards and the datasource as code, so a fresh cluster comes up fully configured.

## Architecture

```
 DCGM exporter (per GPU node)  ──┐
 node-exporter (DaemonSet)     ──┤
 kubelet                       ──┼──►  Prometheus  ──►  Grafana
 cAdvisor (via kubelet)        ──┘    (5s scrape,       (provisioned
                                       30d / 80GB        dashboards)
                                       retention)
```

| Source | Job | Gives us |
|--------|-----|----------|
| NVIDIA DCGM exporter (`gpu-operator` ns) | `dcgm` | GPU utilization, memory, temperature, power, tensor/DRAM/GR-engine activity |
| node-exporter (this repo) | `node-exporter` | Whole-node and per-core CPU, node memory |
| kubelet / cAdvisor | `kubelet`, `cadvisor` | Per-container and per-pod CPU and memory |

All four jobs attach the same `node` label, so CPU, memory and GPU series can be joined and
filtered by node.

## Dashboards

| Folder | Dashboard | Purpose |
|--------|-----------|---------|
| K8s Monitoring | GPU + CPU + Memory | Cluster-wide view per pod, including GPU profiling metrics |
| Node Breakdown | Per-Node Breakdown | One collapsible row per node (13 panels): CPU overall and per core, GPU overall and per GPU index, temperature, power, per-pod usage |
| Cluster Overview | Cluster Overview | One bar per node for CPU, memory, GPU usage, GPU memory, temperature and power; built for presenting |
| Cluster Overview | Cluster Trends | Same metrics as line graphs, one line per node |

## Repository layout

```
monitoring/
  monitoring.yaml     namespace and Prometheus RBAC
  prometheus.yaml     scrape config, 100Gi PVC, deployment, service
  node-exporter.yaml  DaemonSet for node and per-core metrics
  grafana.yaml        datasource, dashboard providers, all dashboards, deployment, service
  RUNBOOK.md          operations and troubleshooting
```

## Deploy

```bash
kubectl apply -f monitoring/monitoring.yaml
kubectl apply -f monitoring/prometheus.yaml
kubectl apply -f monitoring/node-exporter.yaml
kubectl apply -f monitoring/grafana.yaml
```

Prometheus does not reload its config when the ConfigMap changes. After editing scrape jobs, apply
and then reload (`--web.enable-lifecycle` is enabled):

```bash
kubectl apply -f monitoring/prometheus.yaml
curl -X POST http://<prometheus-clusterip>:9090/-/reload
```

Grafana reads dashboards at startup, so restart it after changing them:

```bash
kubectl rollout restart deployment/grafana -n monitoring
```

## Access

Grafana is a NodePort service on `30030`. The nodes are on a private network, so tunnel in:

```bash
ssh -L 3000:<worker-ip>:30030 <user>@<head-node>
```

Open `http://localhost:3000`. The default admin password in the manifest is a placeholder and
must be changed for any real use.

## Operations

See [`monitoring/RUNBOOK.md`](monitoring/RUNBOOK.md) for health checks, storage and retention,
and the Prometheus OOMKill recovery procedure.
