# Redis on Kubernetes — Home Lab Installation Guide

A complete guide to deploying Redis on a self-managed kubeadm Kubernetes cluster using the Bitnami Helm chart, with master/replica architecture, persistent storage, and full validation testing.

> **Author:** Dennis Teimuno  
> **Redis Version:** 8.8.0  
> **Helm Chart Version:** bitnami/redis 27.0.13  
> **Kubernetes Version:** v1.36.0  
> **Namespace:** gitlab  
> **Date:** July 2026

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
  - [Add Bitnami Helm Repo](#add-bitnami-helm-repo)
  - [Create Namespace](#create-namespace)
  - [Redis values.yaml](#redis-valuesyaml)
  - [Deploy Redis](#deploy-redis)
- [Verify Deployment](#verify-deployment)
  - [Check Pods and PVCs](#check-pods-and-pvcs)
  - [Get Redis Password](#get-redis-password)
- [Validation Tests](#validation-tests)
  - [1. Connectivity Test](#1-connectivity-test)
  - [2. Authentication Test](#2-authentication-test)
  - [3. Read Write Test](#3-read-write-test)
  - [4. Replication Health](#4-replication-health)
  - [5. Replica Sync Test](#5-replica-sync-test)
  - [6. Persistence Test](#6-persistence-test)
  - [7. Memory and Stats](#7-memory-and-stats)
  - [8. Performance Benchmark](#8-performance-benchmark)
- [Connecting Applications](#connecting-applications)
  - [GitLab Integration](#gitlab-integration)
  - [Other Applications](#other-applications)
- [Operations](#operations)
  - [Access Redis CLI](#access-redis-cli)
  - [Monitor Redis](#monitor-redis)
  - [Scale Replicas](#scale-replicas)
- [Troubleshooting](#troubleshooting)
- [Redis DNS Reference](#redis-dns-reference)

---

## Overview

This guide deploys Redis inside a Kubernetes cluster using the Bitnami Helm chart. Redis is used as the primary cache and session store for GitLab, but the setup is general enough to serve any application that requires Redis.

**Why Redis on Kubernetes instead of AWS ElastiCache?**

| Factor | Redis on Kubernetes | AWS ElastiCache |
|---|---|---|
| Cost | Free (uses existing cluster) | $0.068/hr minimum |
| Control | Full access, any config | Managed, limited config |
| Stop/start | Stops with EC2 | Always running (always charging) |
| Complexity | Medium | Low |
| Good for | Home lab, dev clusters | Production workloads |

For a home lab with cost constraints, running Redis inside the cluster makes sense — it stops when your EC2 instances stop, so you only pay for the EBS storage when not in use.

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                  gitlab namespace                    │
│                                                      │
│  ┌─────────────────┐    ┌────────────────────────┐  │
│  │  redis-master-0  │───▶│  redis-replicas-0      │  │
│  │  (Read/Write)    │    │  (Read Only)           │  │
│  │  Port 6379       │───▶│  redis-replicas-1      │  │
│  │  8 GiB PVC       │    │  (Read Only)           │  │
│  └─────────────────┘───▶│  redis-replicas-2      │  │
│                          │  (Read Only)           │  │
│                          └────────────────────────┘  │
│                                                      │
│  DNS:                                                │
│  redis-master.gitlab.svc.cluster.local   ← writes   │
│  redis-replicas.gitlab.svc.cluster.local ← reads    │
└─────────────────────────────────────────────────────┘
```

**Components:**

| Component | Count | Purpose |
|---|---|---|
| redis-master | 1 | Handles all write operations |
| redis-replicas | 3 | Handle read operations, provide redundancy |
| PersistentVolumeClaims | 4 | Data persistence across pod restarts |

---

## Prerequisites

- A running Kubernetes cluster (kubeadm, EKS, or any distribution)
- `kubectl` configured and pointing to your cluster
- `helm` v3 installed
- A default StorageClass available

Verify your StorageClass:

```bash
kubectl get storageclass
# Should show a StorageClass marked (default)
```

If no default exists (common on kubeadm), install local-path-provisioner first:

```bash
kubectl apply -f \
  https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml

kubectl patch storageclass local-path \
  -p '{"metadata":{"annotations":
    {"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

---

## Installation

### Add Bitnami Helm Repo

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Verify the chart is available
helm search repo bitnami/redis --versions | head -5
```

### Create Namespace

```bash
kubectl create namespace gitlab

# Verify
kubectl get namespace gitlab
```

### Redis values.yaml

Save the following as `redis-values.yaml`:

```yaml
# redis-values.yaml
# Bitnami Redis Helm Chart — Home Lab Configuration
# Chart version: 27.0.13 | Redis version: 8.8.0

architecture: replication

auth:
  enabled: true
  # Password is auto-generated if not set
  # Retrieve with: kubectl get secret redis -n gitlab
  #   -o jsonpath="{.data.redis-password}" | base64 -d

master:
  count: 1
  persistence:
    enabled: true
    storageClass: local-path
    size: 8Gi
  resources:
    requests:
      memory: 128Mi
      cpu: 100m
    limits:
      memory: 256Mi

replica:
  replicaCount: 3
  persistence:
    enabled: true
    storageClass: local-path
    size: 8Gi
  resources:
    requests:
      memory: 128Mi
      cpu: 100m
    limits:
      memory: 192Mi

metrics:
  enabled: true
  serviceMonitor:
    enabled: false  # Set to true if you have kube-prometheus-stack installed
```

> **Note:** The Bitnami chart auto-generates a strong password and stores it in a Kubernetes secret named after your Helm release. See [Get Redis Password](#get-redis-password) below to retrieve it.

### Deploy Redis

```bash
helm install redis bitnami/redis \
  --version 27.0.13 \
  --namespace gitlab \
  -f redis-values.yaml

# Watch pods come up
kubectl get pods -n gitlab -w
```

Expected output after 1-2 minutes:

```
NAME               READY   STATUS    RESTARTS   AGE
redis-master-0     1/1     Running   0          90s
redis-replicas-0   1/1     Running   0          90s
redis-replicas-1   1/1     Running   0          75s
redis-replicas-2   1/1     Running   0          60s
```

---

## Verify Deployment

### Check Pods and PVCs

```bash
# All pods should be Running
kubectl get pods -n gitlab -l app.kubernetes.io/name=redis

# All PVCs should be Bound
kubectl get pvc -n gitlab

# Check services
kubectl get svc -n gitlab | grep redis
```

Expected PVC output:

```
NAME                       STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS
redis-data-redis-master-0  Bound    pvc-...   8Gi        RWO            local-path
redis-data-redis-replicas-0  Bound  pvc-...   8Gi        RWO            local-path
redis-data-redis-replicas-1  Bound  pvc-...   8Gi        RWO            local-path
redis-data-redis-replicas-2  Bound  pvc-...   8Gi        RWO            local-path
```

Expected service output:

```
NAME              TYPE        CLUSTER-IP      PORT(S)    AGE
redis-headless    ClusterIP   None            6379/TCP   2m
redis-master      ClusterIP   10.x.x.x        6379/TCP   2m
redis-replicas    ClusterIP   10.x.x.x        6379/TCP   2m
```

### Get Redis Password

```bash
# Export to environment variable
export REDIS_PASSWORD=$(kubectl get secret redis \
  -n gitlab \
  -o jsonpath="{.data.redis-password}" | base64 -d)

echo "Redis password: $REDIS_PASSWORD"
```

---

## Validation Tests

Run all validations after deployment to confirm Redis is working correctly.

### 1. Connectivity Test

Verify the master is reachable from within the cluster:

```bash
kubectl run redis-test --rm -it \
  --image=redis:alpine \
  --restart=Never \
  -n gitlab -- \
  redis-cli \
  -h redis-master.gitlab.svc.cluster.local \
  -a $REDIS_PASSWORD \
  ping
```

**Expected output:**
```
PONG
```

### 2. Authentication Test

Verify authentication is enforced — connection without password should fail:

```bash
# Without password — should fail
kubectl run redis-noauth --rm -it \
  --image=redis:alpine \
  --restart=Never \
  -n gitlab -- \
  redis-cli \
  -h redis-master.gitlab.svc.cluster.local \
  ping
```

**Expected output:**
```
NOAUTH Authentication required.
```

### 3. Read Write Test

Verify basic set/get/delete operations:

```bash
# Write a key
kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASSWORD SET testkey "hello-from-lab"

# Read it back
kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASSWORD GET testkey

# Clean up
kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASSWORD DEL testkey
```

**Expected output:**
```
OK
hello-from-lab
(integer) 1
```

### 4. Replication Health

Verify the master has all replicas connected:

```bash
kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASSWORD info replication
```

**Expected output (key fields):**
```
role:master
connected_slaves:3
slave0:ip=...,port=6379,state=online,offset=...,lag=0
slave1:ip=...,port=6379,state=online,offset=...,lag=0
slave2:ip=...,port=6379,state=online,offset=...,lag=0
master_failover_state:no-failover
master_replid:...
```

All 3 replicas should show `state=online`.

### 5. Replica Sync Test

Verify data written to master is readable from replicas:

```bash
# Write to master
kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASSWORD SET replication-test "confirmed"

# Short pause for replication
sleep 1

# Read from each replica
kubectl exec -it redis-replicas-0 -n gitlab -- \
  redis-cli -a $REDIS_PASSWORD GET replication-test

kubectl exec -it redis-replicas-1 -n gitlab -- \
  redis-cli -a $REDIS_PASSWORD GET replication-test

kubectl exec -it redis-replicas-2 -n gitlab -- \
  redis-cli -a $REDIS_PASSWORD GET replication-test

# Clean up from master
kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASSWORD DEL replication-test
```

**Expected output from each replica:**
```
confirmed
```

### 6. Persistence Test

Verify data survives pod restarts (PVC is working):

```bash
# Write a test key
kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASSWORD SET persist-test "survived"

# Delete the master pod (it will be recreated by StatefulSet)
kubectl delete pod redis-master-0 -n gitlab

# Watch it restart
kubectl get pods -n gitlab -w | grep master

# Wait until Running, then check the key survived
kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASSWORD GET persist-test

# Clean up
kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASSWORD DEL persist-test
```

**Expected output after restart:**
```
survived
```

> This confirms your PVC is correctly mounted and data persists across pod lifecycle events.

### 7. Memory and Stats

Check memory usage and persistence settings:

```bash
# Memory usage
kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASSWORD info memory \
  | grep -E "used_memory_human|maxmemory_human|mem_fragmentation_ratio"

# Persistence status
kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASSWORD info persistence \
  | grep -E "rdb_last_save_time|rdb_changes_since_last_save|aof_enabled"

# Server info
kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASSWORD info server \
  | grep -E "redis_version|uptime_in_seconds|tcp_port"
```

### 8. Performance Benchmark

Run a performance test to establish baseline metrics:

```bash
kubectl run redis-bench --rm -it \
  --image=redis:alpine \
  --restart=Never \
  -n gitlab -- \
  redis-benchmark \
  -h redis-master.gitlab.svc.cluster.local \
  -a $REDIS_PASSWORD \
  -n 10000 \
  -c 10 \
  -q
```

**Sample output:**
```
PING_INLINE: 85000 requests per second
PING_MBULK: 87000 requests per second
SET: 82000 requests per second
GET: 89000 requests per second
INCR: 84000 requests per second
LPUSH: 80000 requests per second
RPUSH: 81000 requests per second
```

> Results vary based on your EC2 instance type and network conditions.

---

## Connecting Applications

### GitLab Integration

After deploying Redis, create the secret GitLab expects and reference it in your GitLab values:

```bash
# Create the secret GitLab will reference
kubectl create secret generic redis-password \
  --namespace gitlab \
  --from-literal=password="$REDIS_PASSWORD"
```

Add to your `gitlab-values.yaml`:

```yaml
global:
  redis:
    host: redis-master.gitlab.svc.cluster.local
    port: 6379
    database: 0
    auth:
      enabled: true
      secret: redis-password
      key: password
    sentinelAuth:
      enabled: false
```

### Other Applications

For any other application connecting to Redis from within the cluster:

```yaml
# Environment variables for your application
env:
  - name: REDIS_HOST
    value: "redis-master.gitlab.svc.cluster.local"
  - name: REDIS_PORT
    value: "6379"
  - name: REDIS_PASSWORD
    valueFrom:
      secretKeyRef:
        name: redis-password
        key: password
```

**Connection strings by language:**

```python
# Python (redis-py)
import redis
r = redis.Redis(
    host='redis-master.gitlab.svc.cluster.local',
    port=6379,
    password='YOUR_PASSWORD',
    decode_responses=True
)
```

```javascript
// Node.js (ioredis)
const Redis = require('ioredis');
const redis = new Redis({
  host: 'redis-master.gitlab.svc.cluster.local',
  port: 6379,
  password: 'YOUR_PASSWORD'
});
```

```go
// Go (go-redis)
rdb := redis.NewClient(&redis.Options{
    Addr:     "redis-master.gitlab.svc.cluster.local:6379",
    Password: "YOUR_PASSWORD",
    DB:       0,
})
```

---

## Operations

### Access Redis CLI

```bash
# Interactive session on master
kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASSWORD

# Run a single command
kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASSWORD <command>

# Monitor all commands in real time (useful for debugging)
kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASSWORD MONITOR
```

### Monitor Redis

**Key monitoring commands:**

```bash
# All connected clients
kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASSWORD CLIENT LIST

# Current key count
kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASSWORD DBSIZE

# Slow queries log
kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASSWORD SLOWLOG GET 10

# Real-time stats (updates every second)
kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASSWORD --stat

# Check specific key TTL
kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASSWORD TTL <keyname>
```

### Scale Replicas

To adjust the number of replicas:

```bash
# Scale down to 1 replica (saves memory on a small cluster)
helm upgrade redis bitnami/redis \
  --version 27.0.13 \
  --namespace gitlab \
  --reuse-values \
  --set replica.replicaCount=1

# Scale back to 3
helm upgrade redis bitnami/redis \
  --version 27.0.13 \
  --namespace gitlab \
  --reuse-values \
  --set replica.replicaCount=3
```

---

## Troubleshooting

### Pod Stuck in Pending

```bash
kubectl describe pod redis-master-0 -n gitlab | grep -A 10 Events
kubectl get pvc -n gitlab | grep redis
```

**Common fix:** No default StorageClass. Install local-path-provisioner and set it as default (see [Prerequisites](#prerequisites)).

### Authentication Errors

```bash
# Verify the secret exists and has correct data
kubectl get secret redis -n gitlab -o yaml
kubectl get secret redis -n gitlab \
  -o jsonpath="{.data.redis-password}" | base64 -d
```

### Replicas Not Syncing

```bash
# Check replica logs
kubectl logs redis-replicas-0 -n gitlab --tail=30

# Check master replication status
kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASSWORD info replication \
  | grep -E "connected_slaves|slave"
```

**Common fix:** Ensure the headless service `redis-headless` exists — replicas use it for DNS-based discovery.

### High Memory Usage

```bash
# Check memory usage
kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASSWORD info memory \
  | grep used_memory_human

# Find large keys
kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASSWORD --bigkeys
```

### PVC Full

```bash
# Check PVC usage from inside the pod
kubectl exec -it redis-master-0 -n gitlab -- df -h /data

# Expand the PVC (storage class must support expansion)
kubectl patch pvc redis-data-redis-master-0 -n gitlab \
  -p '{"spec":{"resources":{"requests":{"storage":"16Gi"}}}}'
```

---

## Redis DNS Reference

Use these DNS names from any pod within the cluster:

| DNS Name | Purpose |
|---|---|
| `redis-master.gitlab.svc.cluster.local` | Read/write operations |
| `redis-replicas.gitlab.svc.cluster.local` | Read-only operations |
| `redis-headless.gitlab.svc.cluster.local` | Direct pod access (internal) |

**Port:** `6379` (default Redis port)

**Database index:** `0` (default — GitLab uses database 0)

---

## References

- [Bitnami Redis Helm Chart](https://github.com/bitnami/charts/tree/main/bitnami/redis)
- [Redis Documentation](https://redis.io/docs/)
- [Redis Replication Guide](https://redis.io/docs/management/replication/)
- [GitLab Redis Configuration](https://docs.gitlab.com/charts/charts/globals.html#configure-redis-settings)

---

*Part of the [Self-Managed GitLab on Kubernetes Home Lab](../README.md) project by Dennis Teimuno*
