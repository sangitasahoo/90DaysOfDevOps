# Day 56 - Kubernetes StatefulSets

## Task 1 - Understanding the Problem

### Deployment Manifest

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
```

Commands:

``` bash
kubectl apply -f nginx-deployment.yaml
kubectl get pods
kubectl delete pod <pod-name>
kubectl delete deployment nginx-deployment
```

Observation: - Deployment pods have random names. - Recreated pods get
different names.

**Answer:** Random pod names are unsuitable for database clusters
because node identity changes after recreation, disrupting replication
and clustering.

------------------------------------------------------------------------

## Task 2 - Headless Service

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless
spec:
  clusterIP: None
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

Commands:

``` bash
kubectl apply -f nginx-headless-service.yaml
kubectl get svc
```

Observation: - `CLUSTER-IP` = `None`

**Answer:** Headless Service (`clusterIP: None`) creates DNS records for
each pod instead of load balancing.

------------------------------------------------------------------------

## Task 3 - StatefulSet

``` yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: nginx-headless
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: web-data
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: web-data
    spec:
      accessModes: [ReadWriteOnce]
      resources:
        requests:
          storage: 100Mi
```

Commands:

``` bash
kubectl apply -f nginx-statefulset.yaml
kubectl get pods -l app=nginx -w
kubectl get pvc
```

Observation: - Ordered pod creation: web-0 → web-1 → web-2 - PVCs: -
web-data-web-0 - web-data-web-1 - web-data-web-2

------------------------------------------------------------------------

## Task 4 - Stable Network Identity

Commands:

``` bash
kubectl run busybox --image=busybox:1.36 --restart=Never -it --rm -- sh
nslookup web-0.nginx-headless.default.svc.cluster.local
nslookup web-1.nginx-headless.default.svc.cluster.local
nslookup web-2.nginx-headless.default.svc.cluster.local
kubectl get pods -o wide
```

**Answer:** The IP returned by `nslookup` matches the pod IP.

------------------------------------------------------------------------

## Task 5 - Stable Storage

Commands:

``` bash
kubectl exec web-0 -- sh -c "echo 'Data from web-0' > /usr/share/nginx/html/index.html"
kubectl delete pod web-0
kubectl exec web-0 -- cat /usr/share/nginx/html/index.html
```

Observation: - Data remains after pod recreation because the same PVC is
reattached.

**Answer:** Yes, the data is identical after pod recreation.

------------------------------------------------------------------------

## Task 6 - Ordered Scaling

Commands:

``` bash
kubectl scale statefulset web --replicas=5
kubectl scale statefulset web --replicas=3
kubectl get pvc
```

Observation: - Scale up: web-3 → web-4 - Scale down: web-4 → web-3 -
PVCs remain: - web-data-web-0 - web-data-web-1 - web-data-web-2 -
web-data-web-3 - web-data-web-4

**Answer:** 5 PVCs still exist after scaling down because StatefulSets
retain PVCs to preserve data.

## Key Takeaways

-   Stable pod names
-   Ordered deployment/scaling
-   Stable DNS using Headless Service
-   Dedicated PVC per pod
-   Data survives pod recreation
-   PVCs are retained after scale down
