# Monitoring Stack Runbook

Prometheus + Grafana deployed in the `monitoring` namespace on a 9-node L40S Kubernetes cluster.
The NVIDIA DCGM exporter already runs in the `gpu-operator` namespace (one pod per GPU node).

---

## What's deployed

| Component  | Image                      | Purpose                                      |
|------------|----------------------------|----------------------------------------------|
| Prometheus    | `prom/prometheus:v2.51.2`     | Scrapes and stores metrics (30d / 80GB retention, 100Gi PVC) |
| Grafana       | `grafana/grafana:10.4.2`      | Provisioned dashboards in three folders       |
| node-exporter | `prom/node-exporter:v1.8.1`   | DaemonSet: per-node and per-core CPU, node memory |

Prometheus scrapes:
- **dcgm** — GPU utilization, memory, temperature, power, profiling (pod discovery, one target per node)
- **node-exporter** — whole-node and per-core CPU, node memory (pod discovery, hostNetwork)
- **kubelet** — node-level metrics (all nodes, routed via API server)
- **cadvisor** — per-container CPU and memory (all pods, all namespaces)

Every job carries a `node` label so series can be filtered per node.

---

## Accessing Grafana

Grafana is exposed as a NodePort on port `30030`. The cluster nodes are on a private
network (`192.168.50.x`) so you need SSH port forwarding to reach it from your laptop.

### Step 1 — SSH tunnel from your laptop

```bash
ssh -L 3000:192.168.50.1:30030 <your-username>@192.168.50.100
```

Replace `<your-username>` with your actual login. `192.168.50.100` is the head node.
Keep this terminal open — closing it kills the tunnel.

### Step 2 — Open in browser

```
http://localhost:3000
```

**Login:** `admin` / `changeme`

The "K8s — GPU + CPU + Memory" dashboard loads as the home page. The other dashboards are in the
`Node Breakdown` and `Cluster Overview` folders.

---

## Deploying from scratch

```bash
# From the repo root
kubectl apply -f monitoring/monitoring.yaml
kubectl apply -f monitoring/prometheus.yaml
kubectl apply -f monitoring/node-exporter.yaml
kubectl apply -f monitoring/grafana.yaml

# Watch pods come up
kubectl get pods -n monitoring -w
```

---

## Checking health

```bash
# Pod status
kubectl get pods -n monitoring

# prometheus and grafana should be Running 1/1, plus one node-exporter pod per node
# If Prometheus is CrashLoopBackOff see the fix below

# Services
kubectl get svc -n monitoring

# Prometheus targets (are dcgm, node-exporter, kubelet, cadvisor all up?)
kubectl port-forward -n monitoring svc/prometheus 9090:9090
# then open http://localhost:9090/targets
```

---

## Changing config

Prometheus does not pick up ConfigMap changes on its own. After `kubectl apply -f monitoring/prometheus.yaml`:

```bash
curl -X POST http://$(kubectl get svc -n monitoring prometheus -o jsonpath='{.spec.clusterIP}'):9090/-/reload
```

Startup arguments (for example retention) need `kubectl rollout restart deployment/prometheus -n monitoring`.
Dashboard changes need `kubectl rollout restart deployment/grafana -n monitoring`.

---

## Storage and retention

Prometheus keeps 30 days of data, capped at 80GB (`--storage.tsdb.retention.time` and
`--storage.tsdb.retention.size`). The PVC is 100Gi on Longhorn, which allows volume expansion, so
the size cap stays below the disk and old blocks are dropped before it fills.

---

## Fixing Prometheus CrashLoopBackOff (OOMKilled)

This happens when Prometheus can't finish replaying its WAL on startup because the
memory limit is hit. Each crash leaves the WAL larger, making the next crash happen
sooner — a death spiral.

**Symptoms:**
```bash
kubectl describe pod -n monitoring <prometheus-pod> | grep -A 5 "Last State"
# Shows: Reason: OOMKilled / Exit Code: 137
```

**Fix — wipe the WAL (compacted historical blocks are preserved):**

```bash
# 1. Stop Prometheus
kubectl scale deployment prometheus -n monitoring --replicas=0

# 2. Run a cleanup pod on the same PVC
kubectl run -n monitoring prom-cleanup --restart=Never \
  --image=busybox \
  --overrides='{
    "spec": {
      "volumes": [{"name":"data","persistentVolumeClaim":{"claimName":"prometheus-data"}}],
      "containers": [{
        "name": "prom-cleanup",
        "image": "busybox",
        "command": ["sh","-c","rm -rf /prometheus/wal /prometheus/chunks_head && echo DONE"],
        "volumeMounts": [{"name":"data","mountPath":"/prometheus"}],
        "securityContext": {"runAsUser": 65534}
      }]
    }
  }'

# 3. Confirm it printed DONE
kubectl logs -n monitoring prom-cleanup

# 4. Remove the cleanup pod
kubectl delete pod -n monitoring prom-cleanup

# 5. Bring Prometheus back
kubectl scale deployment prometheus -n monitoring --replicas=1

# 6. Watch logs — WAL replay should now take milliseconds, not minutes
kubectl logs -n monitoring -l app=prometheus -f
```

---

## Cluster nodes

| Node              | IP              | Role          |
|-------------------|-----------------|---------------|
| head-u24-ctrlpl   | 192.168.50.100  | control-plane |
| sys1-u24-worker   | 192.168.50.1    | worker        |
| sys2-u24-worker   | 192.168.50.2    | worker        |
| sys3-u24-worker   | 192.168.50.3    | worker        |
| sys4-u24-worker   | 192.168.50.4    | worker        |
| sys5-u24-worker   | 192.168.50.5    | worker        |
| sys6-u24-worker   | 192.168.50.6    | worker        |
| sys7-u24-worker   | 192.168.50.7    | worker        |
| sys8-u24-worker   | 192.168.50.8    | worker        |
| sys9-u24-worker   | 192.168.50.9    | worker (scheduling disabled) |

---

## Dashboards

| Folder           | Dashboard           | Contents |
|------------------|---------------------|----------|
| K8s Monitoring   | GPU + CPU + Memory  | GPU utilization, memory, temperature, power, per-pod CPU and memory, GR engine / DRAM / tensor core activity |
| Node Breakdown   | Per-Node Breakdown  | Repeating row per node: CPU overall and per core, memory %, GPU overall and per index, GPU memory/temp/power, per-pod usage, profiling metrics |
| Cluster Overview | Cluster Overview    | Bar gauges, one bar per node, for presenting |
| Cluster Overview | Cluster Trends      | The same metrics as line graphs, one line per node |

Troubleshooting empty panels:
- GPU panels empty: check whether DCGM labels metrics with `pod` or `exported_pod` and adjust the PromQL filter.
- Per-node panels empty: confirm the `node-exporter` target is `up` at `/targets` and that Prometheus was reloaded after the last config change.
