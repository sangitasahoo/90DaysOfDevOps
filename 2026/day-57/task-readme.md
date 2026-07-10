# Day 57 - Kubernetes Resources & Probes

## Task 1: Resource Requests and Limits

### Objective
Learn how to configure CPU and memory requests and limits for a Pod and understand Kubernetes QoS (Quality of Service) classes.

### Pod Manifest

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

### Apply the Pod

```bash
kubectl apply -f resource-demo.yaml
```

### Verify

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

### What I Learned

- Requests are the minimum resources guaranteed to a Pod.
- Limits are the maximum resources a Pod can consume.
- Scheduler uses **requests** while placing Pods.
- CPU is measured in millicores.
- Memory is measured in Mi (Mebibytes).
- Since requests and limits are different, the Pod gets **Burstable** QoS.
- If requests = limits → Guaranteed.
- If neither is specified → BestEffort.

---

# Task 2: OOMKilled - Exceeding Memory Limits

### Objective

Understand what happens when a container exceeds its memory limit.

### Pod Manifest

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

### Apply

```bash
kubectl apply -f oom-demo.yaml
```

### Watch

```bash
kubectl get pods -w
```

### Verify

```bash
kubectl describe pod oom-demo
```

Output:

```
Reason: OOMKilled
Exit Code: 137
```

### What I Learned

- Memory limits are enforced strictly.
- When a process exceeds the memory limit, Linux kills it.
- Kubernetes reports the reason as **OOMKilled**.
- Exit code **137** means the process received **SIGKILL**.
- CPU is throttled when exceeding its limit, but memory is not—it results in termination.

---

# Task 3: Pending Pod - Requesting Too Much

### Objective

Learn how Kubernetes schedules Pods based on resource requests.

### Pod Manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: huge-request
spec:
  containers:
    - name: nginx
      image: nginx
      resources:
        requests:
          cpu: "100"
          memory: "128Gi"
```

### Apply

```bash
kubectl apply -f huge-request.yaml
```

### Verify

```bash
kubectl get pods
```

Output:

```
Pending
```

Describe:

```bash
kubectl describe pod huge-request
```

Events:

```
Warning  FailedScheduling
0/1 nodes are available:
1 Insufficient cpu,
1 Insufficient memory
```

### What I Learned

- Scheduler checks only resource requests.
- If no node satisfies the requests, the Pod remains **Pending**.
- Scheduler clearly reports why scheduling failed in the Events section.

---

# Task 4: Liveness Probe

### Objective

Learn how Kubernetes detects unhealthy containers and automatically restarts them.

### Pod Manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-demo
spec:
  containers:
    - name: busybox
      image: busybox
      command:
        - sh
        - -c
        - |
          touch /tmp/healthy
          sleep 30
          rm -f /tmp/healthy
          sleep 600
      livenessProbe:
        exec:
          command:
            - cat
            - /tmp/healthy
        periodSeconds: 5
        failureThreshold: 3
```

### Apply

```bash
kubectl apply -f liveness-demo.yaml
```

### Watch

```bash
kubectl get pods -w
```

### Verify

```bash
kubectl describe pod liveness-demo
```

Output:

```
Liveness probe failed

Container will be restarted
```

### What I Learned

- Liveness probes detect unhealthy or stuck containers.
- Probe runs every 5 seconds.
- After 3 consecutive failures, Kubernetes restarts the container.
- Restart count increases in `kubectl get pods`.

---

# Task 5: Readiness Probe

### Objective

Learn how readiness probes control whether a Pod receives traffic.

### Pod Manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-demo
spec:
  containers:
    - name: nginx
      image: nginx
      readinessProbe:
        httpGet:
          path: /
          port: 80
        periodSeconds: 5
        failureThreshold: 3
```

### Create Service

```bash
kubectl expose pod readiness-demo \
--port=80 \
--name=readiness-svc
```

### Verify Endpoints

```bash
kubectl get endpoints readiness-svc
```

Initially:

```
10.x.x.x:80
```

Break the probe:

```bash
kubectl exec readiness-demo -- \
rm /usr/share/nginx/html/index.html
```

After about 15 seconds:

```bash
kubectl get pods
```

Output:

```
READY 0/1
STATUS Running
RESTARTS 0
```

Endpoints:

```bash
kubectl get endpoints readiness-svc
```

Output:

```
<none>
```

### What I Learned

- Readiness probes decide whether a Pod should receive traffic.
- Failure removes the Pod from Service endpoints.
- The Pod continues running.
- The container is **not restarted**.
- Restart count remains 0.

---

# Task 6: Startup Probe

### Objective

Learn how startup probes protect slow-starting applications.

### Pod Manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-demo
spec:
  containers:
    - name: busybox
      image: busybox
      command:
        - sh
        - -c
        - |
          sleep 20
          touch /tmp/started
          sleep 600
      startupProbe:
        exec:
          command:
            - cat
            - /tmp/started
        periodSeconds: 5
        failureThreshold: 12
      livenessProbe:
        exec:
          command:
            - cat
            - /tmp/started
        periodSeconds: 5
        failureThreshold: 3
```

### Apply

```bash
kubectl apply -f startup-demo.yaml
```

### Verify

```bash
kubectl describe pod startup-demo
```

Initially:

```
Startup probe failed
```

After 20 seconds:

```
Startup probe succeeded
```

Liveness probe starts only after the startup probe succeeds.

### What Happens if `failureThreshold` is 2?

- Probe runs every 5 seconds.
- Total startup time allowed = 10 seconds.
- Application needs 20 seconds.
- Kubernetes kills the container before it finishes starting.
- The Pod repeatedly restarts and eventually enters **CrashLoopBackOff**.

### What I Learned

- Startup probes are designed for slow-starting applications.
- While the startup probe is running, liveness and readiness probes are disabled.
- Once the startup probe succeeds, liveness and readiness probes begin.
- A low `failureThreshold` can cause continuous restarts if the application needs more startup time.

---

# Summary

Today I learned:

- How to configure CPU and memory requests and limits.
- The difference between Requests and Limits.
- Kubernetes QoS classes (Guaranteed, Burstable, BestEffort).
- Why containers become **OOMKilled** when memory limits are exceeded.
- How the scheduler keeps Pods in **Pending** when resource requests cannot be satisfied.
- How **Liveness Probes** automatically restart unhealthy containers.
- How **Readiness Probes** remove Pods from Service endpoints without restarting them.
- How **Startup Probes** give slow-starting applications enough time before liveness and readiness checks begin.
- The practical difference between Startup, Liveness, and Readiness probes and when to use each.
