# Day 55 -- Kubernetes Persistent Volumes

## Task 1: emptyDir

-   Created a Pod with an `emptyDir` volume mounted at `/data`.
-   Wrote a timestamp to `/data/message.txt`.
-   Verified the file with `kubectl exec`.
-   Deleted and recreated the Pod.
-   **Result:** The timestamp changed because `emptyDir` data is deleted
    with the Pod.

## Task 2: Static PersistentVolume

Created a PV with: - Capacity: `1Gi` - AccessMode: `ReadWriteOnce` -
Reclaim Policy: `Retain` - `hostPath: /tmp/k8s-pv-data`

Verified:

``` bash
kubectl get pv
```

Status: **Available**

## Task 3: PersistentVolumeClaim

Created a PVC requesting: - Storage: `500Mi` - AccessMode:
`ReadWriteOnce`

Initially the PVC remained **Pending**.

### Error Solved ⭐

**Problem** - PVC stayed in `Pending`.

**Cause** - The cluster had a default StorageClass. - Since the PVC did
not explicitly disable StorageClass usage, Kubernetes attempted dynamic
provisioning instead of binding to the static PV.

**Solution** Added:

``` yaml
storageClassName: ""
```

to the **PVC**.

The PV did not need `storageClassName: ""` because omitting it meant it
had no StorageClass, allowing it to bind successfully.

After applying the change:

``` bash
kubectl get pvc
kubectl get pv
```

Both resources became **Bound**.

## Task 4: Use PVC in a Pod

-   Mounted the PVC at `/data`.
-   Appended text to `/data/message.txt`.
-   Deleted and recreated the Pod.
-   Verified the file still contained data from both Pod runs.

**Learning:** PVC preserves data even after Pod deletion.

## Task 5: StorageClasses

Explored:

``` bash
kubectl get storageclass
kubectl describe storageclass
```

Learned: - Provisioner - Reclaim Policy - Volume Binding Mode - Default
StorageClass

Dynamic provisioning automatically creates PVs for PVCs.

## Task 6: Dynamic Provisioning

-   Created a PVC using the default StorageClass.
-   Used it in a Pod.
-   Verified data could be written.

### Observation

I expected a new PV named `pvc-...`, but it did not appear.

**Reason** An existing matching manual PV can satisfy a PVC request.
When Kubernetes finds a suitable available PV, it binds to it instead of
dynamically provisioning a new one.

To force dynamic provisioning: 1. Delete or avoid the manual PV. 2.
Create only the dynamic PVC. 3. Kubernetes creates a new PV with a
generated name (for example `pvc-<uuid>`).

------------------------------------------------------------------------

# Key Commands

``` bash
kubectl apply -f <file>.yaml
kubectl get pv
kubectl get pvc
kubectl get storageclass
kubectl describe pvc <pvc-name>
kubectl describe storageclass
kubectl exec <pod> -- cat /data/message.txt
kubectl delete pod <pod-name>
```

# Key Concepts

-   `emptyDir` is ephemeral.
-   PersistentVolumes survive Pod deletion.
-   PVCs request storage from PVs.
-   Static provisioning uses manually created PVs.
-   Dynamic provisioning creates PVs automatically through a
    StorageClass.
-   `storageClassName: ""` prevents the PVC from using the default
    StorageClass.
-   `Retain` preserves data after PVC deletion.
-   `Delete` removes dynamically provisioned storage after PVC deletion
    (depending on StorageClass).
