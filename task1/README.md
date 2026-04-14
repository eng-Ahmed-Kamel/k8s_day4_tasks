# Lab 4 — Part 1: Persistent Volumes

> **Topics covered:** hostPath PV · PersistentVolumeClaim · Retain policy · ReadWriteMany · NodeSelector
---
![Topology](task1/topology.png)
---

## Overview

This lab creates a Kubernetes PersistentVolume (PV) backed by a `hostPath`, binds a PVC to it, and deploys 3 Nginx replicas that all mount the same volume — pinned to the same node.

---

## Step 1a — Prepare the hostPath on the Node

Run this on the Kubernetes node (or via SSH into it):

```bash
# Create the directory
sudo mkdir -p /data/nginx

# Write your full name into index.html
echo "Ahmed Mohamed" | sudo tee /data/nginx/index.html

# Verify
cat /data/nginx/index.html
```

---

## Step 1b — Create the PersistentVolume

**File: `nginx-pv.yaml`**

> ⚠️ `storageClassName: ""` is required. Without it, the cluster's default StorageClass (`local-path`) is assigned to the PV, causing the PVC to stay in **Pending**.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany          # multiple pods can mount simultaneously
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""       # disables dynamic provisioning
  hostPath:
    path: /data/nginx
```

```bash
kubectl apply -f nginx-pv.yaml
```

---

## Step 1c — Create the PersistentVolumeClaim

**File: `nginx-pvc.yaml`**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: ""       # must match the PV
  volumeName: nginx-pv       # explicit static binding
```

```bash
kubectl apply -f nginx-pvc.yaml

# Both should show STATUS = Bound
kubectl get pv,pvc
```

### Troubleshooting — PVC stuck in Pending

If the PVC remains `Pending`, the `storageClassName` is mismatched. Fix it by deleting and re-applying:

```bash
kubectl delete pvc nginx-pvc
kubectl delete pv nginx-pv

kubectl apply -f nginx-pv.yaml
kubectl apply -f nginx-pvc.yaml
```

---

## Step 1d — Create the Deployment (3 replicas, same node)

First get your node name:

```bash
kubectl get nodes
```

**File: `nginx-deployment.yaml`**

Replace `<your-node-name>` with the actual node name from the output above.

```yaml
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
      nodeSelector:
        kubernetes.io/hostname: <your-node-name>   # pins all pods to one node
      containers:
        - name: nginx
          image: nginx:alpine
          volumeMounts:
            - name: web-content
              mountPath: /usr/share/nginx/html
      volumes:
        - name: web-content
          persistentVolumeClaim:
            claimName: nginx-pvc
```

```bash
kubectl apply -f nginx-deployment.yaml

# All 3 pods should be Running on the same node
kubectl get pods -o wide
```
---
# output 
![o/p](task1/1.png)

---

## Verification Checklist

| Check | Command | Expected Result |
|---|---|---|
| PV bound | `kubectl get pv nginx-pv` | `STATUS = Bound` |
| PVC bound | `kubectl get pvc nginx-pvc` | `STATUS = Bound` |
| 3 pods running | `kubectl get pods -o wide` | `3/3 Running` |
| All on same node | `kubectl get pods -o wide` | Same node name for all 3 |
| index.html content | `kubectl exec <pod> -- cat /usr/share/nginx/html/index.html` | Your full name |

---

## Key Concepts

| Concept | Explanation |
|---|---|
| `hostPath` | Mounts a directory from the node's filesystem into pods. Data persists as long as the node exists. |
| `ReadWriteMany` | Allows the volume to be mounted by multiple pods simultaneously — required for 3 replicas sharing one PV. |
| `Retain` | When the PVC is deleted, the PV and its data are preserved. Must be manually reclaimed. |
| `storageClassName: ""` | Disables dynamic provisioning so the static PV can bind correctly. |
| `nodeSelector` | Pins all pods to a specific node by matching the `kubernetes.io/hostname` label. |
| `volumeName` | Explicitly binds the PVC to a named PV, bypassing automatic matching. |