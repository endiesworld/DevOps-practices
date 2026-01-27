# Helm

Helm is a package manager for Kubernetes that simplifies the deployment and management of applications on Kubernetes clusters. It uses "charts," which are pre-configured packages of Kubernetes resources, to define, install, and upgrade applications.
## Features
- **Package Management**: Helm allows you to package your Kubernetes applications into charts, making it easy to share and distribute them.
- **Dependency Management**: Helm can manage dependencies between charts, allowing you to define and install complex applications with multiple components.
- **Versioning**: Helm supports versioning of charts, enabling you to easily roll back to previous versions of your applications.
- **Templating**: Helm uses Go templating to create dynamic and reusable Kubernetes manifests, allowing for customization of deployments.
- **Release Management**: Helm tracks the state of your deployments, making it easy to upgrade, rollback, and manage releases.
## Installation
To install Helm, follow these steps:
1. Download the Helm binary from the [official Helm releases page](https://github.com/helm/helm/releases) for your operating system.

2. Extract the binary and move it to a directory in your system's PATH.
3. Verify the installation by running:
```bash
   helm version
```

## Getting Started Helm 3+
To get started with Helm 3+, follow these steps:
1. **Add a Helm Repository**: Add a Helm chart repository to your local Helm client:

```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo update
```

**repo:** Tells Helm you want to perform an action related to Repositories (the "app stores").

**add:** The specific action—you are connecting a new source to your local Helm client.

**prometheus-community:** The Local Aliases. You are giving that specific URL a nickname so you don't have to type the long web address every time you want to install something from it.

**https://...:** The actual URL where the Chart files are hosted on the internet.


2. **Install a Chart**: Install a Helm chart from the repository:
```bash
helm install my-release prometheus-community/kube-prometheus-stack
```
**install:** Tells Helm you want to install a chart.
**my-release:** The name you are giving to this specific installation of the chart. You can choose any name you like.
**prometheus-community/kube-prometheus-stack:** The chart you want to install, specified by its repository alias and chart name.

## Templates & "Dry Runs"
Before we move into the actual installation of an app, it's vital to know how to "test" your configuration. Since Helm templates are dynamic, it's easy to make a typo that results in invalid YAML.

We use the debug and dry-run flags to see what Helm would send to Kubernetes without actually doing it:
```bash
helm install my-app ./my-chart --dry-run --debug
```
This prints the final, filled-in YAML files to your screen. It’s the best way to verify that your values.yaml are being injected into the templates correctly.


## CRDs (Custom Resource Definitions)
Some Helm charts include Custom Resource Definitions (CRDs) that extend the Kubernetes API. When installing charts with CRDs, you may need to use the `--skip-crds` flag to avoid conflicts with existing CRDs:

```bash
kubectl get crds
```
This command lists all the CRDs currently installed in your Kubernetes cluster. If the chart you are installing includes CRDs that are already present in the cluster, you can use the following command to skip installing them:
```bash
helm install my-release my-chart --skip-crds
```
This flag tells Helm to skip installing the CRDs defined in the chart, allowing you to manage them separately if needed.

### Prometheus Example with CRDs
When installing the Prometheus Operator chart, which includes CRDs, you can use the following command:
```bash
helm show crds prometheus-community/kube-prometheus-stack \
| kubectl apply --server-side -f -
helm upgrade --install prometheus-1 prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace

```


## Monitoring Your Helm Releases
To monitor and manage your Helm releases, you can use the following commands:
- List all releases:
```bash
helm list
```
- Upgrade a release:
```bash
helm upgrade my-release prometheus-community/kube-prometheus-stack --values custom-values.yaml
```
- Rollback a release to a previous version:
```bash
helm rollback my-release 1
```
- Uninstall a release:
```bash
helm uninstall my-release
``` 
