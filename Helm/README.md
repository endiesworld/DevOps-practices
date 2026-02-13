# Helm: Zero to Mastery for Absolute Beginners

This guide is for people who are new to Kubernetes and Helm.

Goal: help you start from zero, understand the core ideas, and confidently deploy, upgrade, debug, and package real apps with Helm.

## 1. What Is Helm?

Helm is a package manager for Kubernetes.

Without Helm, you often manage many YAML files manually:
- `Deployment`
- `Service`
- `Ingress`
- `ConfigMap`
- `Secret`
- storage resources

Helm groups those resources into a reusable package called a **Chart**.

With Helm, you can:
- Install apps with one command
- Upgrade safely
- Roll back to older versions
- Reuse templates and values across environments

## 2. Core Terms You Must Know

- **Chart**: A package of Kubernetes manifests + templates.
- **Release**: A running instance of a chart in your cluster.
- **Repository (repo)**: A place where charts are published.
- **Values**: Configuration inputs for templates (`values.yaml` + overrides).
- **Template**: Kubernetes YAML with Go-template placeholders.

Example:
- Chart name: `kube-prometheus-stack`
- Release name: `monitoring`
- Namespace: `monitoring`

## 3. Prerequisites

Install and verify:
- Kubernetes cluster (`kind`, `minikube`, or cloud cluster)
- `kubectl`
- `helm`

```bash
kubectl version --client
helm version
kubectl get nodes
```

## 4. Install Helm

Use the official docs for your OS:
- https://helm.sh/docs/intro/install/

Verify:

```bash
helm version
```

## 5. First Hands-On: Install Your First Chart

### Step 1: Add a chart repository

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### Step 2: Search available charts

```bash
helm search repo prometheus-community
```

### Step 3: Install a chart

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

### Step 4: Check what Helm installed

```bash
helm list -A
kubectl get all -n monitoring
```

## 6. Most Important Daily Commands

```bash
# List releases
helm list -A

# Show release history
helm history monitoring -n monitoring

# Upgrade a release
helm upgrade monitoring prometheus-community/kube-prometheus-stack -n monitoring

# Roll back to revision 1
helm rollback monitoring 1 -n monitoring

# Uninstall release
helm uninstall monitoring -n monitoring
```

## 7. Values: How Configuration Works

When Helm installs a chart, it merges values in this order (low to high priority):

1. Chart default `values.yaml`
2. `-f file1.yaml`
3. `-f file2.yaml` (later file overrides earlier)
4. `--set key=value`

Inspect chart defaults:

```bash
helm show values prometheus-community/kube-prometheus-stack
```

Install with your custom values:

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  -f values-common.yaml \
  -f values-prod.yaml
```

Quick one-off override:

```bash
helm upgrade monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --set grafana.enabled=false
```

Inspect applied values:

```bash
helm get values monitoring -n monitoring
helm get values monitoring -n monitoring --all
```

## 8. Dry Run and Template Debugging (Critical Skill)

Always preview before applying in important environments:

```bash
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f values-prod.yaml \
  --dry-run --debug
```

Render templates locally:

```bash
helm template monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f values-prod.yaml > rendered.yaml
```

Validate rendered YAML with Kubernetes API:

```bash
kubectl apply --dry-run=server -f rendered.yaml
```

## 9. Build Your Own Chart

Create chart skeleton:

```bash
helm create myapp
cd myapp
```

Key files:
- `Chart.yaml`: chart metadata and version
- `values.yaml`: default config
- `templates/`: Kubernetes templates
- `templates/_helpers.tpl`: naming helpers

Lint your chart:

```bash
helm lint .
```

Test render:

```bash
helm template myapp .
```

Install local chart:

```bash
helm install myapp-dev . -n dev --create-namespace
```

## 10. Template Basics You Need

Example template snippet:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "myapp.name" . }}
```

Useful patterns:
- `{{ .Values.key }}`: read from values
- `{{ if ... }} ... {{ end }}`: conditionals
- `{{ range ... }} ... {{ end }}`: loops
- `| default "value"`: fallback
- `| quote`: force string quoting

## 11. Environment Strategy (Dev, Staging, Prod)

Use separate values files:
- `values.yaml` (base)
- `values-dev.yaml`
- `values-staging.yaml`
- `values-prod.yaml`

Example:

```bash
helm upgrade --install myapp . \
  -n myapp-prod \
  -f values.yaml \
  -f values-prod.yaml
```

Keep environment differences in values, not template logic.

## 12. Helm + CRDs

**CRD** means **Custom Resource Definition**.

Kubernetes has built-in resource types like:
- `Pod`
- `Deployment`
- `Service`

A CRD lets you add a **new resource type** to Kubernetes.

Example:
- Built-in: `Deployment`
- Custom (from Prometheus Operator): `ServiceMonitor`

### Why CRDs are needed

Some platforms need domain-specific objects that do not exist in core Kubernetes.

For monitoring stacks, teams need objects like:
- `ServiceMonitor`
- `Prometheus`
- `Alertmanager`

Those are not native Kubernetes kinds, so the chart installs CRDs to teach the API server those new kinds.

### How CRDs work (simple model)

1. You install a CRD (new schema/type) into the cluster.
2. Kubernetes API now recognizes that new `kind`.
3. You can create custom resources of that kind.
4. A controller/operator watches those resources and turns them into real cluster behavior.

Without step 1, applying those custom resources fails because Kubernetes does not know that type.

### Why this matters in Helm

Charts that depend on custom kinds must have CRDs installed first (or already present), otherwise parts of the release can fail.

That is why CRDs are handled carefully in production upgrades.

Check CRDs in a chart:

```bash
helm show crds prometheus-community/kube-prometheus-stack
```

Common approach for mature environments:
1. Apply CRDs explicitly.
2. Install/upgrade chart.

```bash
helm show crds prometheus-community/kube-prometheus-stack | kubectl apply -f -
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace
```

## 13. Package and Share Charts

Package a chart:

```bash
helm package .
```

This produces `myapp-<version>.tgz`.

You can host packaged charts in:
- GitHub Pages
- ChartMuseum
- OCI registries (recommended modern approach)

Push to OCI registry:

```bash
helm registry login <registry>
helm package .
helm push myapp-<version>.tgz oci://<registry>/<repo>
```

Install from OCI:

```bash
helm install myapp oci://<registry>/<repo>/myapp --version <version>
```

## 14. Troubleshooting Playbook

When a release fails:

```bash
helm status <release> -n <ns>
helm history <release> -n <ns>
helm get manifest <release> -n <ns>
kubectl get events -n <ns> --sort-by=.lastTimestamp
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns>
```

Most common causes:
- Wrong image/tag
- Bad values key path
- Missing secret/config
- Resource limits too strict
- CRD/version mismatch

## 15. Security and Best Practices

- Never commit raw secrets in `values.yaml`
- Use secret managers or encrypted secret workflows
- Pin chart versions in production
- Use `helm lint` and `--dry-run --debug` in CI
- Review rendered manifests before production
- Prefer small, explicit values files
- Keep chart versions meaningful (`MAJOR.MINOR.PATCH`)

## 16. Learning Roadmap (Zero to Mastery)

### Level 1: Beginner (Day 1-2)
- Install Helm
- Add repo, search, install chart
- Use `helm list`, `helm status`, `helm uninstall`

### Level 2: Intermediate (Day 3-5)
- Override values with `-f` and `--set`
- Use `helm upgrade --install`
- Use `--dry-run --debug`
- Understand rollback with `helm history` + `helm rollback`

### Level 3: Advanced (Week 2)
- Build your own chart with `helm create`
- Use conditionals/loops/helpers in templates
- Structure multiple environment values files
- Handle CRDs safely

### Level 4: Production Ready (Week 3+)
- Package and publish charts
- Use OCI registries
- Add lint/template checks in CI/CD
- Define versioning and release strategy

## 17. Quick Practice Labs

1. Install NGINX chart and change service type to `NodePort`.
2. Deploy same chart to `dev` and `prod` with different replica counts.
3. Break a template intentionally, catch it with `helm lint`.
4. Upgrade release, then roll back.
5. Package your chart and install from packaged `.tgz`.

## 18. Cheat Sheet

```bash
helm repo add <name> <url>
helm repo update
helm search repo <keyword>

helm install <release> <chart> -n <ns> --create-namespace
helm upgrade --install <release> <chart> -n <ns> -f values.yaml
helm rollback <release> <revision> -n <ns>
helm uninstall <release> -n <ns>

helm show values <chart>
helm template <release> <chart> -f values.yaml
helm lint <chart-dir>

helm get values <release> -n <ns>
helm get manifest <release> -n <ns>
helm history <release> -n <ns>
```

## 19. Final Advice

Mastering Helm is mostly about repetition:
- Install
- Inspect
- Change values
- Dry-run
- Upgrade
- Roll back

If you can do that loop confidently, you are production-capable with Helm.
