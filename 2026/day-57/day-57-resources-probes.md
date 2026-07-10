# Day 57 - Kubernetes Resources & Probes

## Objective

Learn how Kubernetes manages CPU and memory resources, how Quality of Service (QoS) classes are assigned, what happens when containers exceed resource limits, and how health probes (Liveness, Readiness, and Startup) keep applications healthy and available.

---

# 1. Resource Requests and Limits

## What are Resource Requests?

A **request** is the minimum amount of CPU and memory that Kubernetes guarantees for a container.

The Kubernetes Scheduler uses requests to decide which node has enough available resources to run the Pod.

Example:

```yaml
requests:
  cpu: "100m"
  memory: "128Mi"
```

Meaning:

- CPU: 100 millicores (0.1 CPU)
- Memory: 128 MiB

---

## What are Resource Limits?

A **limit** is the maximum amount of CPU or memory a container can use.

If the container exceeds:

- CPU limit → CPU gets throttled.
- Memory limit → Container is immediately killed (OOMKilled).

Example:

```yaml
limits:
  cpu: "250m"
  memory: "256Mi"
```

Meaning:

- Maximum CPU = 0.25 CPU
- Maximum Memory = 256 MiB

---

## Pod Manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "250m"
        memory: "256Mi"
```

Apply:

```bash
kubectl apply -f resource-demo.yaml
```

Inspect:

```bash
kubectl describe pod resource-demo
```

Output:

```
Requests:
  cpu: 100m
  memory: 128Mi

Limits:
  cpu: 250m
  memory: 256Mi

QoS Class: Burstable
```

---

## QoS (Quality of Service)

Kubernetes assigns every Pod one of three QoS classes.

### 1. Guaranteed

Requirements:

- CPU request = CPU limit
- Memory request = Memory limit

Example:

```yaml
requests:
  cpu: "500m"
  memory: "256Mi"

limits:
  cpu: "500m"
  memory: "256Mi"
```

Highest priority during node pressure.

---

### 2. Burstable

Requests and limits are present but different.

Example:

```yaml
requests:
  cpu: 100m

limits:
  cpu: 250m
```

This was today's example.

---

### 3. BestEffort

No requests or limits specified.

Example:

```yaml
resources: {}
```

Lowest priority.

These Pods are the first to be evicted when the node runs out of resources.

---

# CPU vs Memory

CPU:

- Can exceed request.
- If above limit → Kubernetes throttles CPU.

Memory:

- Can exceed request.
- If above limit → Linux OOM Killer terminates the process.

CPU is throttled.

Memory is killed.

---

# 2. OOMKilled (Out Of Memory)

## Goal

Understand what happens when a container exceeds its memory limit.

Manifest:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: oom-demo
spec:
  containers:
  - name: stress
    image: polinux/stress
    command: ["stress"]
    args:
    - "--vm"
    - "1"
    - "--vm-bytes"
    - "200M"
    - "--vm-hang"
    - "1"
    resources:
      limits:
        memory: "100Mi"
```

Apply:

```bash
kubectl apply -f oom-demo.yaml
```

Watch:

```bash
kubectl get pods -w
```

Describe:

```bash
kubectl describe pod oom-demo
```

Output:

```
Reason: OOMKilled
Exit Code: 137
```

---

## Why Exit Code 137?

```
137 = 128 + 9
```

- 128 → Process exited because of a signal.
- 9 → SIGKILL.

The Linux kernel kills the process because it exceeded its memory limit.

---

# 3. Pending Pod (Insufficient Resources)

## Goal

Learn how the scheduler behaves when a Pod requests more resources than any node has.

Manifest:

```yaml
resources:
  requests:
    cpu: "100"
    memory: "128Gi"
```

Apply:

```bash
kubectl apply -f huge-request.yaml
```

Check:

```bash
kubectl get pods
```

Status:

```
Pending
```

Describe:

```bash
kubectl describe pod huge-request
```

Events:

```
Warning FailedScheduling

0/1 nodes are available:
1 Insufficient cpu,
1 Insufficient memory
```

The scheduler refuses to place the Pod because no node satisfies the requested resources.

---

# Kubernetes Scheduler

Scheduler uses only **requests**.

It ignores limits while deciding where to place a Pod.

Flow:

```
Pod Created
      │
      ▼
Scheduler checks Requests
      │
Enough resources?
      │
 ┌────┴────┐
 │         │
Yes        No
 │         │
 ▼         ▼
Running   Pending
```

---

# 4. Liveness Probe

Purpose:

Detect containers that are stuck or unhealthy.

If the probe fails repeatedly:

Kubernetes restarts the container.

Manifest:

```yaml
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  periodSeconds: 5
  failureThreshold: 3
```

Container startup:

```
touch /tmp/healthy

sleep 30

rm /tmp/healthy
```

Timeline:

```
0 sec
Healthy

↓

30 sec

File deleted

↓

Probe fails

↓

5 sec

Failure #1

↓

10 sec

Failure #2

↓

15 sec

Failure #3

↓

Restart
```

Observe:

```bash
kubectl get pod -w
```

Restart count increases.

---

# 5. Readiness Probe

Purpose:

Control whether a Pod receives traffic.

Readiness failure:

- Pod keeps running.
- Container is NOT restarted.
- Pod removed from Service endpoints.

Manifest:

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
```

Expose service:

```bash
kubectl expose pod readiness-demo \
--port=80 \
--name=readiness-svc
```

Check endpoints:

```bash
kubectl get endpoints readiness-svc
```

Initially:

```
10.244.x.x:80
```

Break readiness:

```bash
kubectl exec readiness-demo -- \
rm /usr/share/nginx/html/index.html
```

After about 15 seconds:

```
READY 0/1
STATUS Running
RESTARTS 0
```

Endpoints:

```
<none>
```

Container is never restarted.

---

# Liveness vs Readiness

| Liveness | Readiness |
|----------|-----------|
| Detects dead containers | Detects whether app can receive traffic |
| Failure restarts container | Failure removes Pod from Service |
| Container restarts | Container keeps running |
| Used for recovery | Used for traffic control |

---

# 6. Startup Probe

Purpose:

Allow slow-starting applications enough time to initialize.

While Startup Probe is running:

- Liveness Probe is disabled.
- Readiness Probe is disabled.

Manifest:

```yaml
startupProbe:
  exec:
    command:
    - cat
    - /tmp/started
  periodSeconds: 5
  failureThreshold: 12
```

Container:

```
sleep 20

touch /tmp/started
```

Budget:

```
12 × 5 = 60 seconds
```

Application starts in:

```
20 seconds
```

Therefore startup succeeds.

Only then does the liveness probe begin.

---

## What if failureThreshold = 2?

Budget:

```
2 × 5 = 10 seconds
```

Application needs:

```
20 seconds
```

Result:

```
Probe fails

↓

Container killed

↓

Restart

↓

Probe fails again

↓

CrashLoopBackOff
```

---

# Probe Comparison

| Probe | Purpose | On Failure |
|--------|---------|------------|
| Startup Probe | Wait for slow startup | Restart container if startup never completes |
| Liveness Probe | Check if container is alive | Restart container |
| Readiness Probe | Check if container is ready for traffic | Remove Pod from Service endpoints |

---

# Important kubectl Commands

Apply resources:

```bash
kubectl apply -f file.yaml
```

Watch Pods:

```bash
kubectl get pods -w
```

Describe Pod:

```bash
kubectl describe pod <pod-name>
```

View restart count:

```bash
kubectl get pod
```

View events:

```bash
kubectl describe pod
```

Check Service endpoints:

```bash
kubectl get endpoints
```

Create Service:

```bash
kubectl expose pod <pod-name> --port=80 --name=<service-name>
```

Execute command inside Pod:

```bash
kubectl exec <pod-name> -- <command>
```

---

# Key Takeaways

- **Requests** are the minimum resources guaranteed to a container and are used by the scheduler for Pod placement.
- **Limits** define the maximum resources a container can consume.
- CPU exceeding its limit is **throttled**, while memory exceeding its limit causes an **OOMKilled** event.
- QoS classes determine eviction priority:
  - **Guaranteed** → Highest priority.
  - **Burstable** → Medium priority.
  - **BestEffort** → Lowest priority.
- Pods remain **Pending** if no node can satisfy their resource requests.
- **Liveness Probe** restarts unhealthy containers.
- **Readiness Probe** controls whether a Pod receives traffic without restarting it.
- **Startup Probe** protects slow-starting applications by delaying liveness and readiness checks until startup completes.
- Understanding requests, limits, QoS, and probes is essential for building reliable, production-ready Kubernetes workloads.
