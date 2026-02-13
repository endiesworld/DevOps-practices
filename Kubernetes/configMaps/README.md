# Kubernetes ConfigMaps Runbook

This runbook explains the main ways to consume ConfigMap values and when to use each approach.

## 1. Scope

1. ConfigMaps are for non-sensitive configuration.
2. Secrets are for sensitive values.
3. A ConfigMap is namespaced and limited to 1 MiB total size.

## 2. Choose the Right Consumption Pattern

| Pattern | How it works | Use when | Update behavior |
|---|---|---|---|
| `env` + `value` (literal) | Hardcode value directly in manifest | Value is static and not shared across workloads | Manifest change + rollout needed |
| `env` + `valueFrom.configMapKeyRef` | Map specific keys to specific env var names | You need strict control of which keys are exposed or want to rename env vars | Pod restart required |
| `envFrom.configMapRef` | Import all ConfigMap keys as env vars | App expects many env vars and key names already match app env names | Pod restart required |
| Volume mount (`volumes[].configMap`) | Each key appears as a file | App reads config files from disk (`.properties`, `.yaml`, `.json`) | Usually auto-refreshes in mounted dir |
| Volume mount with `subPath` | Mount one file from ConfigMap | App requires one exact file path | No auto-refresh; restart required |

## 3. Prerequisites

```bash
kubectl version --client
kubectl config current-context
kubectl get ns
kubectl get ns stevens
```

## 4. Create a Base ConfigMap

`configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: stevens
data:
  APP_MODE: "dev"
  LOG_LEVEL: "info"
  FEATURE_X_ENABLED: "true"
  app.properties: |
    server.port=8080
    feature.x=true
```

Apply and verify:

```bash
kubectl apply -f configmap.yaml
kubectl -n stevens get configmap app-config
kubectl -n stevens describe configmap app-config
```

## 5. Pattern A: `env` + `valueFrom` (Selective Key Mapping)

Use this when you want explicit, per-key mapping and control.

`env` is the container env list. Inside each env entry, you either set:
1. `value` for a literal value, or
2. `valueFrom` to pull from a ConfigMap/Secret/field reference.

`deploy-env-valuefrom.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cfg-env-valuefrom
  namespace: stevens
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cfg-env-valuefrom
  template:
    metadata:
      labels:
        app: cfg-env-valuefrom
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "sleep 3600"]
        env:
        - name: APPLICATION_MODE
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_MODE
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL
```

Apply and verify:

```bash
kubectl apply -f deploy-env-valuefrom.yaml
kubectl -n stevens rollout status deployment/cfg-env-valuefrom
kubectl -n stevens exec deployment/cfg-env-valuefrom -- printenv | grep -E "APPLICATION_MODE|LOG_LEVEL"
```

When to use:
1. You only need a few keys.
2. You need env var renaming (`APP_MODE` -> `APPLICATION_MODE`).
3. You want clear ownership of each env var in manifest review.

## 6. Pattern B: `envFrom` (Bulk Import to Environment)

Use this when you want all keys available as env vars with minimal YAML.

`deploy-envfrom.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cfg-envfrom
  namespace: stevens
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cfg-envfrom
  template:
    metadata:
      labels:
        app: cfg-envfrom
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "sleep 3600"]
        envFrom:
        - configMapRef:
            name: app-config
```

Apply and verify:

```bash
kubectl apply -f deploy-envfrom.yaml
kubectl -n stevens rollout status deployment/cfg-envfrom
kubectl -n stevens exec deployment/cfg-envfrom -- printenv | grep -E "APP_MODE|LOG_LEVEL|FEATURE_X_ENABLED"
```

When to use:
1. Key names already match what your application expects.
2. You want less manifest boilerplate.
3. You accept that all keys become env vars.

Watch-outs:
1. Invalid env var key names are skipped.
2. Key collisions across multiple `envFrom` sources can cause confusion.

## 7. Pattern C: Mount ConfigMap as Files (Volume)

Use this when your app reads files rather than env vars.

For a beginner-first explanation of `volume` vs `volumeMount` beyond ConfigMaps, see `Kubernetes/volumes/README.md`.

`deploy-volume.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cfg-volume
  namespace: stevens
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cfg-volume
  template:
    metadata:
      labels:
        app: cfg-volume
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "sleep 3600"]
        volumeMounts:
        - name: cfg
          mountPath: /etc/app-config
      volumes:
      - name: cfg
        configMap:
          name: app-config
          items:
          - key: app.properties
            path: app.properties
```

Apply and verify:

```bash
kubectl apply -f deploy-volume.yaml
kubectl -n stevens rollout status deployment/cfg-volume
kubectl -n stevens exec deployment/cfg-volume -- ls -l /etc/app-config
kubectl -n stevens exec deployment/cfg-volume -- cat /etc/app-config/app.properties
```

When to use:
1. Application expects config files.
2. You need multi-line config format fidelity.
3. You want Kubernetes-managed refresh behavior for mounted config directory.

## 8. `subPath` Variant (Single File Mount)

Use only when the app requires one exact file location and cannot use a mounted directory.

```yaml
volumeMounts:
- name: cfg
  mountPath: /app/config/app.properties
  subPath: app.properties
volumes:
- name: cfg
  configMap:
    name: app-config
```

Important:
1. `subPath` mounts do not auto-refresh when ConfigMap changes.
2. You must restart Pods to pick up updates.

## 9. Update and Rollout Behavior

1. `env` + `valueFrom`: no live refresh, restart required.
2. `envFrom`: no live refresh, restart required.
3. Volume mount directory: updates usually appear automatically within a short sync window.
4. Volume mount with `subPath`: restart required.

Example update and restart:

```bash
kubectl -n stevens create configmap app-config \
  --from-literal=APP_MODE=prod \
  --from-literal=LOG_LEVEL=debug \
  --from-literal=FEATURE_X_ENABLED=true \
  --from-file=app.properties=./app.properties \
  -o yaml --dry-run=client | kubectl apply -f -

kubectl -n stevens rollout restart deployment/cfg-env-valuefrom
kubectl -n stevens rollout restart deployment/cfg-envfrom
kubectl -n stevens rollout status deployment/cfg-env-valuefrom
kubectl -n stevens rollout status deployment/cfg-envfrom
```

## 10. Troubleshooting

ConfigMap missing:

```bash
kubectl -n stevens get configmap
kubectl -n stevens describe configmap app-config
```

Pod not receiving expected env vars:

```bash
kubectl -n stevens get deploy cfg-env-valuefrom cfg-envfrom -o yaml | grep -nE "envFrom|configMapKeyRef|name:"
kubectl -n stevens exec deployment/cfg-env-valuefrom -- printenv | grep -E "APPLICATION_MODE|LOG_LEVEL"
kubectl -n stevens exec deployment/cfg-envfrom -- printenv | grep -E "APP_MODE|LOG_LEVEL|FEATURE_X_ENABLED"
```

Volume mount appears empty:

```bash
kubectl -n stevens describe pod -l app=cfg-volume
kubectl -n stevens exec deployment/cfg-volume -- ls -la /etc/app-config
```

## 11. Cleanup

```bash
kubectl -n stevens delete deployment cfg-env-valuefrom cfg-envfrom cfg-volume --ignore-not-found
kubectl -n stevens delete configmap app-config --ignore-not-found
```

## 12. Related Guides

1. Kubernetes overview: `Kubernetes/README.md`
2. Volumes and volumeMounts (beginner): `Kubernetes/volumes/README.md`
3. Port forwarding runbook: `Kubernetes/Port-forwarding.md`
