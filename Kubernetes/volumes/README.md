# Kubernetes Volumes and VolumeMounts (Beginner Guide)

This guide explains `volumes` and `volumeMounts` from first principles, then walks through practical examples.

## 1. Core Idea

In Kubernetes, storage is defined in two places:

1. `spec.volumes` (Pod level): defines the storage source.
2. `spec.containers[].volumeMounts` (Container level): defines where that storage appears inside a container.

Think of it as:
1. `volume` = what the storage is.
2. `volumeMount` = where the container sees it.

## 2. Why This Matters

Container filesystems are ephemeral by default. Data can be lost when a Pod is replaced. Volumes provide controlled data handling for:

1. Temporary shared data between containers.
2. Injecting config files.
3. Persisting app data across Pod restarts/reschedules.

## 3. Quick Decision Guide

| Use case | Volume type | Data survives Pod restart? | Typical example |
|---|---|---|---|
| Scratch/temp/shared within one Pod | `emptyDir` | No | Cache, intermediate files |
| Config files from Kubernetes objects | `configMap` | N/A (config source) | `app.properties`, YAML config |
| Sensitive files (certs, tokens) | `secret` | N/A (secret source) | TLS certs, API keys |
| Persistent application data | `persistentVolumeClaim` | Yes | Databases, uploads |
| Node-local path (mostly dev/single-node) | `hostPath` | Depends on node | Local testing only |

## 4. Prerequisites

```bash
kubectl version --client
kubectl config current-context
kubectl get ns
kubectl get ns stevens
```

All examples below use namespace `stevens`.

## 5. Anatomy of a Minimal Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo
  namespace: stevens
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: data-volume
      mountPath: /data
  volumes:
  - name: data-volume
    emptyDir: {}
```

Important:
1. `volumeMounts[].name` must exactly match one `volumes[].name`.
2. `mountPath` must be an absolute path.

## 6. Example A: `emptyDir` (Ephemeral Shared Storage)

Use this when multiple containers in the same Pod need to share temporary files.

`emptydir-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-emptydir-demo
  namespace: stevens
spec:
  containers:
  - name: writer
    image: busybox:1.36
    command: ["sh", "-c", "while true; do date > /shared/time.txt; sleep 2; done"]
    volumeMounts:
    - name: shared-data
      mountPath: /shared
  - name: reader
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /shared
  volumes:
  - name: shared-data
    emptyDir: {}
```

Apply and verify:

```bash
kubectl apply -f emptydir-pod.yaml
kubectl -n stevens get pod shared-emptydir-demo
kubectl -n stevens exec shared-emptydir-demo -c reader -- cat /shared/time.txt
kubectl -n stevens describe pod shared-emptydir-demo
```

What you learn:
1. Both containers mount the same volume.
2. Data is shared within the Pod.
3. Data is deleted when the Pod is deleted.

## 7. Example B: ConfigMap as Volume (Files on Disk)

Use this when the app reads config files from disk.

`app-config-volume.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-files
  namespace: stevens
data:
  app.properties: |
    server.port=8080
    feature.flag=true
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config-file-demo
  namespace: stevens
spec:
  replicas: 1
  selector:
    matchLabels:
      app: config-file-demo
  template:
    metadata:
      labels:
        app: config-file-demo
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "sleep 3600"]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/app-config
      volumes:
      - name: config-volume
        configMap:
          name: app-config-files
```

Apply and verify:

```bash
kubectl apply -f app-config-volume.yaml
kubectl -n stevens rollout status deployment/config-file-demo
kubectl -n stevens exec deployment/config-file-demo -- ls -l /etc/app-config
kubectl -n stevens exec deployment/config-file-demo -- cat /etc/app-config/app.properties
```

## 8. Example C: PVC Volume (Persistent Data)

Use this when data must survive Pod restarts and rescheduling.

`pvc-demo.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: demo-data-pvc
  namespace: stevens
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pvc-demo
  namespace: stevens
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pvc-demo
  template:
    metadata:
      labels:
        app: pvc-demo
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "sleep 3600"]
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: demo-data-pvc
```

Apply and verify:

```bash
kubectl apply -f pvc-demo.yaml
kubectl -n stevens get pvc demo-data-pvc
kubectl -n stevens rollout status deployment/pvc-demo
kubectl -n stevens exec deployment/pvc-demo -- sh -c 'echo "hello" > /data/hello.txt'
kubectl -n stevens exec deployment/pvc-demo -- cat /data/hello.txt
```

Persistence check (recreate Pod):

```bash
kubectl -n stevens rollout restart deployment/pvc-demo
kubectl -n stevens rollout status deployment/pvc-demo
kubectl -n stevens exec deployment/pvc-demo -- cat /data/hello.txt
```

## 9. Example D: `hostPath` (Node Filesystem Mount)

`hostPath` mounts a directory or file from the Kubernetes node into your Pod.

Use cases:
1. Local development on single-node clusters (Minikube, Kind, single-node k3s).
2. Node-level agents that read host logs or runtime files.
3. Specialized workloads that must access a known path on the host.

Avoid for general app data in production:
1. It tightly couples Pods to node filesystem layout.
2. It can create security risk by exposing host files to containers.
3. Data behavior varies when Pods move to different nodes.

### `hostPath` vs `PVC` (Quick Comparison)

| Aspect | `hostPath` | `PersistentVolumeClaim (PVC)` |
|---|---|---|
| Backing storage | Direct path on a node filesystem | Abstract claim bound to a PersistentVolume via StorageClass |
| Portability across nodes | Low | High |
| Best fit | Local/dev labs, node-agent workloads | Stateful apps, shared team environments, production paths |
| Scheduling impact | Pod is effectively tied to node/path availability | Scheduler can place Pod where bound volume is accessible |
| Data persistence | Persists only on that node/path | Persists according to storage backend policy |
| Security risk | Higher (container can touch host filesystem) | Lower and more controlled |
| Multi-node reliability | Weak | Stronger and designed for this |
| Recommended default | No (except specific node-level needs) | Yes |

`hostpath-demo.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hostpath-demo
  namespace: stevens
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hostpath-demo
  template:
    metadata:
      labels:
        app: hostpath-demo
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "sleep 3600"]
        volumeMounts:
        - name: host-data
          mountPath: /host-data
      volumes:
      - name: host-data
        hostPath:
          path: /tmp/k8s-hostpath-demo
          type: DirectoryOrCreate
```

Apply and verify:

```bash
kubectl apply -f hostpath-demo.yaml
kubectl -n stevens rollout status deployment/hostpath-demo
kubectl -n stevens exec deployment/hostpath-demo -- sh -c 'echo "from-hostpath" > /host-data/hello.txt'
kubectl -n stevens exec deployment/hostpath-demo -- cat /host-data/hello.txt
kubectl -n stevens get pod -l app=hostpath-demo -o wide
```

Important:
1. Prefer `readOnly: true` on `volumeMounts` when write access is not required.
2. `DirectoryOrCreate` is safer for demos because Kubernetes creates the path if missing.
3. For persistent production storage, use PVC-backed volumes instead of `hostPath`.

## 10. `subPath` (Common Confusion)

`subPath` mounts one file/folder from a volume to a specific path:

```yaml
volumeMounts:
- name: config-volume
  mountPath: /app/config/app.properties
  subPath: app.properties
```

Use carefully:
1. Good when app needs one exact file path.
2. ConfigMap updates do not auto-refresh with `subPath`.
3. Usually requires Pod restart to pick up changes.

## 11. Common Beginner Mistakes

1. Name mismatch between `volumes[].name` and `volumeMounts[].name`.
2. Missing namespace for PVC/ConfigMap.
3. Using relative `mountPath` instead of absolute path.
4. Expecting `emptyDir` data to survive Pod deletion.
5. Expecting `env`-based or `subPath`-based config to live-refresh.
6. Using `hostPath` for production app data on multi-node clusters.

## 12. Troubleshooting Commands

```bash
kubectl -n stevens get pods,pvc,configmap
kubectl -n stevens describe pod <pod-name>
kubectl -n stevens logs <pod-name>
kubectl -n stevens exec <pod-name> -- mount
kubectl -n stevens exec <pod-name> -- ls -la /data
kubectl -n stevens describe pod -l app=hostpath-demo
```

If a volume does not mount:
1. Check `kubectl describe pod <pod-name>` events first.
2. Verify referenced object exists in same namespace (`ConfigMap`, `Secret`, `PVC`).
3. For `hostPath`, verify the node path and file permissions are valid.
4. Confirm RBAC/policies if cluster restrictions are enabled.

## 13. Cleanup

```bash
kubectl -n stevens delete pod shared-emptydir-demo --ignore-not-found
kubectl -n stevens delete deployment config-file-demo pvc-demo hostpath-demo --ignore-not-found
kubectl -n stevens delete configmap app-config-files --ignore-not-found
kubectl -n stevens delete pvc demo-data-pvc --ignore-not-found
```

Note: `hostPath` data remains on the node filesystem (`/tmp/k8s-hostpath-demo`) until removed on that node.

## 14. Related Guides

1. Kubernetes overview: `Kubernetes/README.md`
2. ConfigMaps guide: `Kubernetes/configMaps/README.md`
3. Port forwarding runbook: `Kubernetes/Port-forwarding.md`
4. Storage exercise: `Kubernetes/storage/README.md`
