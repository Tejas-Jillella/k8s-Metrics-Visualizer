# k8s Metrics Visualizer

Real-time GPU + CPU + Memory monitoring for a 9-node L40S Kubernetes cluster.

## Stack

- **Prometheus** — scrapes DCGM (GPU), kubelet, and cAdvisor (CPU/memory) across all nodes and namespaces
- **Grafana** — live dashboard, updates every 5 seconds, 7 days retention
- **DCGM** — already running in `gpu-operator` namespace, provides GPU metrics

## Deploying the monitoring stack

```bash
kubectl apply -f monitoring/00-namespace.yaml
kubectl apply -f monitoring/01-prometheus-rbac.yaml
kubectl apply -f monitoring/02-prometheus-config.yaml
kubectl apply -f monitoring/03-prometheus.yaml
kubectl apply -f monitoring/04-grafana-config.yaml
kubectl apply -f monitoring/05-grafana-dashboard.yaml
kubectl apply -f monitoring/06-grafana.yaml
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
