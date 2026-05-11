# Monitoring Stack Runbook

Prometheus + Grafana deployed in the `monitoring` namespace on a 9-node L40S Kubernetes cluster.
DCGM is already running in the `gpu-operator` namespace at `10.98.78.19:9400`.

---

## What's deployed

| Component  | Image                      | Purpose                                      |
|------------|----------------------------|----------------------------------------------|
| Prometheus | `prom/prometheus:v2.51.2`  | Scrapes DCGM, kubelet, cAdvisor              |
| Grafana    | `grafana/grafana:10.4.2`   | Dashboard — GPU + CPU + Memory for all pods  |
| Redis      | existing                   | Used by the Go collector (separate from this) |

Prometheus scrapes:
- **DCGM** — GPU utilization, memory, temperature, power (static target `10.98.78.19:9400`)
- **kubelet** — node-level metrics (all 9 nodes, routed via API server)
- **cAdvisor** — per-container CPU & memory (all pods, all namespaces)

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

The "K8s — GPU + CPU + Memory" dashboard loads as the home page.

---

## Deploying from scratch

```bash
# From the repo root
kubectl apply -f monitoring/00-namespace.yaml
kubectl apply -f monitoring/01-prometheus-rbac.yaml
kubectl apply -f monitoring/02-prometheus-config.yaml
kubectl apply -f monitoring/03-prometheus.yaml
kubectl apply -f monitoring/04-grafana-config.yaml
kubectl apply -f monitoring/05-grafana-dashboard.yaml
kubectl apply -f monitoring/06-grafana.yaml

# Watch pods come up
kubectl get pods -n monitoring -w
```

---

## Checking health

```bash
# Pod status
kubectl get pods -n monitoring

# Both should show Running 1/1
# If Prometheus is CrashLoopBackOff see the fix below

# Services
kubectl get svc -n monitoring

# Prometheus targets (are DCGM, kubelet, cadvisor all green?)
# Open in browser via SSH tunnel:
http://localhost:9090/targets
# (change the -L port to 9090:192.168.50.1:30090 in your SSH tunnel, or use port-forward)
kubectl port-forward -n monitoring svc/prometheus 9090:9090
```

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

## Dashboard panels

| Panel                          | Metric                                  |
|--------------------------------|-----------------------------------------|
| GPU Utilization (%)            | `DCGM_FI_DEV_GPU_UTIL`                 |
| GPU Memory Used (MiB)          | `DCGM_FI_DEV_FB_USED`                  |
| GPU Temperature (°C)           | `DCGM_FI_DEV_GPU_TEMP`                 |
| GPU Power Usage (W)            | `DCGM_FI_DEV_POWER_USAGE`              |
| CPU Usage — All Pods           | `container_cpu_usage_seconds_total`     |
| Memory Working Set — All Pods  | `container_memory_working_set_bytes`    |

If GPU panels are empty, check whether DCGM is labeling metrics with `pod` or
`exported_pod`. Update the PromQL filter accordingly:
```
# If pod label is missing, try:
DCGM_FI_DEV_GPU_UTIL{exported_pod!=""}
```
