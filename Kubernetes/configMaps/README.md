# ConfigMaps in Kubernetes
This document provides an overview of ConfigMaps in Kubernetes, explaining their purpose, usage, and best practices for managing configuration data in your Kubernetes applications.

## What is a ConfigMap?
A ConfigMap is a Kubernetes object that allows you to store non-confidential configuration data in key-value pairs. ConfigMaps enable you to decouple configuration artifacts from image content, making it easier to manage application configurations across different environments. When a pod is created, inject the configmap into the pod so the key value pairs are available as environment variables or as configuration files in a volume.

There are two phases involved in configuring Configmaps:
1. Creating a ConfigMap
2. Using a ConfigMap in a Pod

## Creating a ConfigMap
There are two ways of creating a configMapp:
1. The imperative way without using a config map definition file and 

```bash
kubectl create configmap my-config --from-literal=key1=value1 --from-literal=key2=value2
```
**Explanation:** This command creates a ConfigMap named `my-config` with two key-value pairs: `key1=value1` and `key2=value2`.
If you wish to add additional key value pairs, simply specify the from literal options multiple times.

2. The declarative way using a config map definition file (YAML or JSON)
Create a file named `configmap.yaml` with the following content:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  key1: value1
  key2: value2
```
Then apply the file using kubectl:
```bash
kubectl apply -f configmap.yaml
```

## Viewing ConfigMaps
To view the created ConfigMaps, use the following command:
```bash
kubectl get configmaps
kubectl describe configmap my-config    
```

## Using a ConfigMap in a Pod
You can use a ConfigMap in a Pod in two ways:
1. As environment variables
2. As configuration files in a volume
### 1. Using ConfigMap as Environment Variables
Create a Pod definition file named `pod-env.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: nginx
    envFrom:
    - configMapRef:
        name: my-config
```
Then create the Pod using:
```bash
kubectl apply -f pod-env.yaml
```
### 2. Using ConfigMap as Configuration Files in a Volume
Create a Pod definition file named `pod-volume.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: nginx
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: my-config
```
Then create the Pod using:
```bash
kubectl apply -f pod-volume.yaml
```