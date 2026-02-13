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

## 9. `subPath` (Common Confusion)

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

## 10. Common Beginner Mistakes

1. Name mismatch between `volumes[].name` and `volumeMounts[].name`.
2. Missing namespace for PVC/ConfigMap.
3. Using relative `mountPath` instead of absolute path.
4. Expecting `emptyDir` data to survive Pod deletion.
5. Expecting `env`-based or `subPath`-based config to live-refresh.

## 11. Troubleshooting Commands

```bash
kubectl -n stevens get pods,pvc,configmap
kubectl -n stevens describe pod <pod-name>
kubectl -n stevens logs <pod-name>
kubectl -n stevens exec <pod-name> -- mount
kubectl -n stevens exec <pod-name> -- ls -la /data
```

If a volume does not mount:
1. Check `kubectl describe pod <pod-name>` events first.
2. Verify referenced object exists in same namespace (`ConfigMap`, `Secret`, `PVC`).
3. Confirm RBAC/policies if cluster restrictions are enabled.

## 12. Cleanup

```bash
kubectl -n stevens delete pod shared-emptydir-demo --ignore-not-found
kubectl -n stevens delete deployment config-file-demo pvc-demo --ignore-not-found
kubectl -n stevens delete configmap app-config-files --ignore-not-found
kubectl -n stevens delete pvc demo-data-pvc --ignore-not-found
```

## 13. Related Guides

1. Kubernetes overview: `Kubernetes/README.md`
2. ConfigMaps guide: `Kubernetes/configMaps/README.md`
3. Port forwarding runbook: `Kubernetes/Port-forwarding.md`
4. Storage exercise: `Kubernetes/storage/README.md`
