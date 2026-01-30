# Monitoring

This directory contains instructions and configurations for setting up monitoring in your Kubernetes cluster using the Prometheus Operator and Grafana.

## CRDs Required

**What are CRDs?**

- Recall that the default kubernets resources (like Pods, Services, Deployments, etc.) may not cover all the needs for specialized applications.

Custom Resource Definitions (CRDs) are a Kubernetes feature that allows you to define custom resources. In the context of monitoring, CRDs are used by the Prometheus Operator to define custom monitoring resources like `Prometheus`, `ServiceMonitor`, and `Alertmanager`.
Before installing the Prometheus Operator, ensure that the necessary CRDs are applied to your cluster. You can find the required CRDs in the `crds/` directory or install them directly using Helm.

**Defining a CRD:**
A CRD is defined using a YAML file that specifies the schema and behavior of the custom resource. Here is a simple example of a CRD definition for a custom resource called `MyResource`
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: myresources.example.com
spec:
  group: example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                field1:
                  type: string
                field2:
                  type: integer
  scope: Namespaced
  names:
    plural: myresources
    singular: myresource
    kind: MyResource
    shortNames:
      - mr
```

**Some Common CRDs for Monitoring:**
- `Prometheus`: Defines a Prometheus instance.
- `ServiceMonitor`: Specifies how to monitor services.
- `Alertmanager`: Configures Alertmanager instances.


## How to view CRDs installed in your cluster
To view the CRDs installed in your Kubernetes cluster, you can use the following command:
```bash
kubectl get crds

kubectl get customresourcedefinitions.apiextensions.k8s.io
```


## Install Prometheus Operator and Grafana using Helm
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus-1 prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

## Extract Default Monitoring Stack using Helm
```bash
helm show values prometheus-community/kube-prometheus-stack > Monitoring/prometheus-default-values.yaml
```

Get the actual Grafana password (the correct way)

Since your release name is prometheus-1 and namespace is monitoring, run:
```bash
kubectl -n monitoring get secret | grep grafana

# You should see something like prometheus-1-grafana.
# Now decode the password:

kubectl -n monitoring get secret prometheus-1-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

Login with: username: admin and the password printed above.


