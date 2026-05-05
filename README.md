# Senior SRE/DevOps Challenge: Designing and Deploying a Resilient, Backed-Up Stateful Application on Kubernetes

---

## High-Level Architecture / Design

### Architecture Overview

This solution is designed around sharded application pods running as Kubernetes StatefulSets.

<img width="761" height="801" alt="ninox drawio" src="https://github.com/user-attachments/assets/6461012d-7c9c-4d85-b0ac-0b387bdf5bb0" />


Each shard owns:
- A dedicated application pod
- Its own LevelDB instance
- A dedicated 2TB LVM-backed persistent volume

Traffic is routed using consistent hashing.

### Request Flow

```text
Client
  ↓
Ingress (Caddy / NGINX / Envoy)
  ↓
Shard Router (consistent hashing)
  ↓
StatefulSet Pods (Shard-0, Shard-1, ...)
  ↓
LevelDB
  ↓
LVM-backed Storage
  ↓
Restic Backups
```

---

## Kubernetes Cluster

- Kubernetes StatefulSets used for stable pod identity and storage
- Headless service for direct shard communication
- Local NVMe-backed LVM storage
- One PVC per pod (2TB)

Example pod DNS:

```text
ninox-0.ninox-headless
ninox-1.ninox-headless
ninox-2.ninox-headless
```

---

## Pipelines

CI/CD pipeline is implemented using GitHub Actions.

### Flow

```text
Git Push
   ↓
GitHub Actions
   ↓
Build Docker Image
   ↓
Push to Registry
   ↓
Helm Deploy to Kubernetes
```

---

## Monitoring & Alerting

### Stack

- Prometheus
- Grafana
- Loki
- Alertmanager

### Monitored Metrics

- Pod restart count
- Node health
- PVC disk usage
- Backup success/failure
- Restore success/failure
- Request latency

### Alerts

- Backup older than 6h
- Disk usage > 80%
- Pod crash loops
- Node unavailable

---

## Security

- RBAC with least privilege
- Kubernetes Secrets / Vault
- Network Policies
- TLS termination via Caddy or Envoy
- Image scanning using Trivy

---

# Implementation / POC

## StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: ninox

spec:
  serviceName: ninox-headless
  replicas: 3

  selector:
    matchLabels:
      app: ninox

  template:
    metadata:
      labels:
        app: ninox

    spec:
      containers:
      - name: app
        image: ninox/app:latest

        volumeMounts:
        - name: data
          mountPath: /data

  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      storageClassName: lvm-storage
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 2Ti
```

---

## Backup Process

Backup is done using LVM snapshots + Restic.

### Steps

```bash
lvcreate -L10G -s -n snap /dev/vg/data
mount /dev/vg/snap /mnt/snap
restic backup /mnt/snap
umount /mnt/snap
lvremove -f /dev/vg/snap
```

### Backup Frequency

- Every 6 hours
- RPO = 6 hours

---

## Horizontal Scaling Strategy (Not Traditional Autoscaling)

Since LevelDB is single-node, traditional autoscaling is not possible.

Scaling is achieved via sharding.

### Horizontal Scaling

1. Add new pod (new shard)
2. Update consistent hash ring
3. Start routing new traffic
4. Run background rebalancing

### Requirements

- Controlled shard addition
- Data migration (if required)

---

## Vertical Scaling

Pods can be vertically scaled by increasing:

- CPU
- Memory

Benefits:
- Better LevelDB compaction
- Improved read/write performance

---

## Auto-healing

Handled natively by Kubernetes.

### Failure Handling

- Pod crash → restarted automatically
- Node failure → pod rescheduled
- Liveness probe restarts unhealthy pods

---

# Recovery and Performance

## Restore Process (Automated)

If pod starts with empty volume:

1. Init container runs restore

```bash
restic restore latest --target /data
```

2. Data restored
3. Application starts
4. Readiness probe passes

---

## Recovery Timeline (Approximate)

| Step | Time |
|---|---|
| Node failure detection | ~5 min |
| Scheduling | ~30 sec |
| Restore 2TB | 40–90 min |
| Total recovery | 60–90 min |

---

## Performance Optimizations

### Storage
- Local NVMe disks
- LVM snapshots

### Backup
- Incremental Restic backups

### Restore
- Parallel restore jobs

```bash
restic restore --jobs 8
```

### Architecture
- Sharding reduces blast radius
- Smaller restore units (2TB per shard)

---

# Final Notes

This design intentionally prioritizes:

- Operational simplicity
- Predictable recovery
- Strong backups
- Fast restore

Instead of introducing complex distributed storage or replication layers.

Core principle:

> Accept failures as normal and recover reliably.
