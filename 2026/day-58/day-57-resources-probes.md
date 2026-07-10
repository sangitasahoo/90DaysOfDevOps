# Day 57 -- Kubernetes Resources & Probes

## Topics Covered

-   Resource Requests and Limits
-   QoS Classes (BestEffort, Burstable, Guaranteed)
-   OOMKilled and Exit Code 137
-   Pending Pods due to insufficient resources
-   Liveness Probe
-   Readiness Probe
-   Startup Probe

## Resource Requests vs Limits

-   Requests reserve minimum CPU/Memory for scheduling.
-   Limits define the maximum runtime usage.
-   CPU exceeding limits is throttled.
-   Memory exceeding limits results in OOMKilled.

## QoS Classes

### BestEffort

-   No requests or limits.

### Burstable

-   Requests and limits are set but differ.

### Guaranteed

-   Requests equal limits for CPU and memory.

## OOMKilled

-   Memory limit: 100Mi
-   Stress allocated: 200M
-   Result: OOMKilled
-   Exit Code: 137 (128 + SIGKILL)

## Pending Pod

-   Requested 100 CPUs and 128Gi memory.
-   Pod stayed Pending.
-   Scheduler event:
    -   FailedScheduling
    -   Insufficient CPU
    -   Insufficient Memory

## Liveness Probe

-   Detects unhealthy containers.
-   Failed probe restarts the container.
-   Demo: deleting /tmp/healthy caused automatic restart.

## Readiness Probe

-   Controls whether a Pod receives traffic.
-   Failed readiness removes Pod from Service endpoints.
-   Container keeps running; it is not restarted.

## Startup Probe

-   Gives slow-starting applications extra startup time.
-   Liveness and readiness probes remain disabled until startup
    succeeds.
-   With periodSeconds=5 and failureThreshold=12, startup budget = 60
    seconds.
-   If failureThreshold=2, the container restarts before finishing
    startup and can enter CrashLoopBackOff.

## Commands Practiced

``` bash
kubectl apply -f <file>.yaml
kubectl describe pod <pod-name>
kubectl get pods -w
kubectl get endpoints <service-name>
kubectl expose pod <pod-name> --port=80 --name=<service-name>
kubectl exec <pod-name> -- rm /usr/share/nginx/html/index.html
```

## Key Takeaways

-   Scheduler uses resource requests for placement.
-   Kubelet enforces resource limits.
-   Memory over limit = OOMKilled.
-   CPU over limit = throttled.
-   Liveness restarts containers.
-   Readiness only controls traffic.
-   Startup probes protect slow-starting applications.
