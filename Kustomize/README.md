# Kustomize: Zero to Mastery for Absolute Beginners

This guide helps you go from zero knowledge to production-ready Kustomize usage.

Goal: learn how to manage Kubernetes YAML cleanly across multiple environments without duplicating files.

## 1. What Is Kustomize?

Kustomize is a Kubernetes-native configuration tool.

It lets you:
- Keep a common **base** set of YAML manifests
- Create environment-specific **overlays** (dev, staging, prod)
- Apply targeted changes (image tags, replicas, labels, patches)

Unlike Helm, Kustomize does not use templates by default.  
You work with plain YAML and compose changes on top.

## 2. Why Kustomize Is Needed

Without Kustomize, teams often copy/paste YAML for each environment:
- `deployment-dev.yaml`
- `deployment-staging.yaml`
- `deployment-prod.yaml`

That creates drift and mistakes.

Kustomize solves this by:
1. Defining one reusable base.
2. Layering only differences per environment.
3. Building final manifests automatically.

## 3. Core Terms You Must Know

- **Base**: Shared manifests used by all environments.
- **Overlay**: Environment-specific customization of a base.
- **kustomization.yaml**: The control file that tells Kustomize what to build.
- **Patch**: A focused change applied to a resource.
- **Transformer**: Built-in modification like `namespace`, labels, prefixes.
- **Generator**: Creates `ConfigMap` or `Secret` from files/literals.

## 4. Prerequisites

- Kubernetes cluster (`kind`, `minikube`, or cloud)
- `kubectl`
- Optional standalone `kustomize` CLI

Verify:

```bash
kubectl version --client
kubectl kustomize --help
```

If you installed standalone CLI:

```bash
kustomize version
```

## 5. Your First Kustomize Project

Create a simple app:

```bash
mkdir -p app/base
cd app/base
```

`deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
      - name: app
        image: nginx:1.25
        ports:
        - containerPort: 80
```

`service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-app
spec:
  selector:
    app: demo-app
  ports:
  - port: 80
    targetPort: 80
```

`kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

Build manifests:

```bash
kubectl kustomize .
```

Apply to cluster:

```bash
kubectl apply -k .
```

## 6. Base and Overlays (Most Important Concept)

Recommended structure:

```text
app/
  base/
    deployment.yaml
    service.yaml
    kustomization.yaml
  overlays/
    dev/
      kustomization.yaml
    prod/
      kustomization.yaml
```

`overlays/dev/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
nameSuffix: -dev
namespace: dev
replicas:
  - name: demo-app
    count: 1
images:
  - name: nginx
    newTag: "1.25.5"
```

`overlays/prod/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
nameSuffix: -prod
namespace: prod
replicas:
  - name: demo-app
    count: 3
images:
  - name: nginx
    newTag: "1.27.0"
```

Apply each environment:

```bash
kubectl apply -k overlays/dev
kubectl apply -k overlays/prod
```

## 7. Patches: Change Only What You Need

Use patches for targeted edits instead of duplicating entire files.

`overlays/prod/patch-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
spec:
  template:
    spec:
      containers:
      - name: app
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
```

Add patch in `overlays/prod/kustomization.yaml`:

```yaml
patches:
  - path: patch-deployment.yaml
```

## 8. ConfigMap and Secret Generators

Generate ConfigMaps from literals/files:

```yaml
configMapGenerator:
  - name: app-config
    literals:
      - LOG_LEVEL=info
      - FEATURE_X=true
```

Generate Secret:

```yaml
secretGenerator:
  - name: app-secret
    literals:
      - DB_PASSWORD=change-me
```

Important: avoid committing real secrets to Git. Use external secret systems in production.

## 9. Useful Transformers

In `kustomization.yaml`, you can apply global updates:

- `namespace: my-ns`
- `namePrefix: team1-`
- `nameSuffix: -prod`
- `commonLabels`
- `commonAnnotations`

Example:

```yaml
commonLabels:
  app.kubernetes.io/managed-by: kustomize
  env: prod
```

## 10. Replacements and Advanced Composition

Use `replacements` when one field should copy from another resource field (for example service name into an env var).  
Use **components** for reusable feature blocks (for example "enable monitoring", "enable network policy") shared across overlays.

These are advanced tools; use them after base/overlay/patches are solid.

## 11. Kustomize in GitOps

Kustomize works well with GitOps tools:
- Flux
- Argo CD

Typical flow:
1. Commit base and overlays to Git.
2. GitOps controller watches repo.
3. Controller runs Kustomize build/apply to the cluster.
4. Git becomes the source of truth.

## 12. Debugging and Validation Workflow

Preview final YAML:

```bash
kubectl kustomize overlays/prod
```

Save output:

```bash
kubectl kustomize overlays/prod > rendered-prod.yaml
```

Server-side validation:

```bash
kubectl apply --dry-run=server -f rendered-prod.yaml
```

Diff before apply:

```bash
kubectl diff -k overlays/prod
```

## 13. Kustomize vs Helm (When to Use Which)

Use Kustomize when:
- You want plain YAML without template language.
- You manage your own manifests.
- You want clean environment overlays.

Use Helm when:
- You install third-party packaged apps.
- You need chart ecosystem and release history.
- You need advanced templating and dependency management.

Many teams use both: Helm for vendor apps, Kustomize for in-house apps.

## 14. Best Practices

- Keep base minimal and reusable.
- Keep overlays small and focused.
- Prefer patches over full-file duplication.
- Keep one responsibility per overlay.
- Validate with `kubectl diff -k` before apply.
- Treat secrets carefully (never plain credentials in Git).

## 15. Learning Roadmap (Zero to Mastery)

### Level 1: Beginner (Day 1-2)
- Build/apply one `kustomization.yaml`
- Understand resources list
- Use `kubectl apply -k`

### Level 2: Intermediate (Day 3-5)
- Create `base` + `dev/prod` overlays
- Change image tag and replicas per environment
- Use simple patches

### Level 3: Advanced (Week 2)
- Use generators and transformers
- Use replacements/components
- Add validation and diff checks in CI

### Level 4: Production Ready (Week 3+)
- Integrate with Flux/Argo CD
- Standardize directory conventions
- Add policy checks and security controls

## 16. Practice Labs

1. Create one base and two overlays (`dev`, `prod`).
2. Set replicas to `1` in dev and `3` in prod.
3. Patch prod with CPU/memory limits.
4. Add a config map generator and consume it in deployment.
5. Run `kubectl diff -k` before applying each change.

## 17. Cheat Sheet

```bash
# Render manifests
kubectl kustomize <path>

# Apply manifests
kubectl apply -k <path>

# Delete manifests
kubectl delete -k <path>

# Diff manifests
kubectl diff -k <path>

# Save rendered output
kubectl kustomize <path> > out.yaml

# Validate rendered output
kubectl apply --dry-run=server -f out.yaml
```

## 18. Final Advice

Master Kustomize by repeating this loop:
1. Build a base.
2. Add overlays.
3. Apply patch-based changes.
4. Preview/diff before apply.
5. Promote the same base safely across environments.

If you can do this consistently, you are production-capable with Kustomize.
