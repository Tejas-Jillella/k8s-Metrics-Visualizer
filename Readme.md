# k8s Metrics Visualizer

Real-time GPU + CPU + Memory monitoring for a 9-node L40S Kubernetes cluster.

## Stack

- **Prometheus** — scrapes DCGM (GPU), node-exporter (per-node/per-core CPU + memory), kubelet, and cAdvisor (per-container CPU/memory) across all nodes and namespaces
- **Grafana** — two dashboards, updating every 5 seconds, 30 days retention:
  - `K8s Monitoring` folder — cluster-wide GPU/CPU/memory dashboard
  - `Node Breakdown` folder — per-node dashboard with CPU % (overall + per-core) and GPU % (overall + per-GPU-index)
- **DCGM** — already running in `gpu-operator` namespace, provides GPU metrics
- **node-exporter** — DaemonSet deployed by this repo, provides per-node/per-core CPU and memory metrics

## Deploying the monitoring stack

```bash
kubectl apply -f monitoring/monitoring.yaml
kubectl apply -f monitoring/prometheus.yaml
kubectl apply -f monitoring/node-exporter.yaml
kubectl apply -f monitoring/grafana.yaml
```

## Accessing Grafana

SSH tunnel from your laptop (keep this terminal open):

```bash
ssh -L 3000:192.168.50.1:30030 <your-username>@192.168.50.100
```

Then open `http://localhost:3000` — login `admin` / `changeme`.

## Go collector

`kube-metrics.go` is a Go-based collector that polls kubelet and logs CPU/memory
metrics to Redis. It predates the Prometheus stack and is kept for reference.

## Runbook

See [`monitoring/RUNBOOK.md`](monitoring/RUNBOOK.md) for health checks, troubleshooting,
and the Prometheus OOMKill fix procedure.
