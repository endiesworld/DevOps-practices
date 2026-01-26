# Kubernetes

## Pod Creation Example
Pods are the smallest deployable units in Kubernetes. Here are some example commands to create and manage pods:

```bash
kubectl run -h | less # Display help information for the 'kubectl run' command "/dry-run" option

kubectl run myapp --image=myimage --dry-run=client -o yaml > myapp-deployment.yaml

kubectl apply -f myapp-deployment.yaml # Apply the generated YAML file to create the deployment. There is also create option.

kubectl get pods # Verify that the pod is created

kubectl describe pod myapp | less # Get detailed information about the created pod. Use the JKL keys to navigate up and down.

kubectl exec -it nginx-emmanuel -- /bin/bash # Access the pod's shell interactively

apt install iputils-ping # Install ping utility inside the pod

ping google.com # Test network connectivity from within the pod

exit # Exit the pod's shell

kubectl logs myapp # View the logs of the created pod

kubectl delete pod myapp # Clean up by deleting the created pod
```

## Deployment Example
Deployments in Kubernetes are used to manage a set of identical pods. Here are some example commands to create and manage deployments:

```bash
Examples:
  # Create a deployment named my-dep that runs the busybox image
  kubectl create deployment my-dep --image=busybox

  # Create a deployment with a command
  kubectl create deployment my-dep --image=busybox -- date

  # Create a deployment named my-dep that runs the nginx image with 3 replicas
  kubectl create deployment my-dep --image=nginx --replicas=3

  # Create a deployment named my-dep that runs the busybox image and expose port 5701
  kubectl create deployment my-dep --image=busybox --port=5701

  # Create a deployment named my-dep that runs multiple containers
  kubectl create deployment my-dep --image=busybox:latest --image=ubuntu:latest --image=nginx

  watch -n 1 "kubectl get pods" # Watch the pods status
```

## Namespace Management
In Kubernetes, namespaces are used to organize and manage resources within a cluster. Here are some common commands for managing namespaces:

```bash
kubectl create namespace my-namespace # Create a new namespace

kubectl create namespace stevens --dry-run=client -o yaml > stevens-ns.yaml  # Generate YAML for namespace creation

kubectl get namespaces # List all namespaces

kubectl get ns # Short form to list all namespaces

kubectl describe namespace my-namespace | less # Get detailed information about a specific namespace

kubectl get pods --namespace=my-namespace # List all pods in a specific namespace
kubectl get pods -n my-namespace # Short form to list all pods in a specific namespace

kubectl run myapp --image=myimage --namespace=my-namespace # Create a pod in a specific namespace

kubectl config set-context --current --namespace=my-namespace # Set the default namespace for the current context
kubectl config view | less # View the current kubeconfig settings

kubectl port-forward pods/mealie-5df8c97467-xl67x 9000 -n stevens # Port forward to a pod in a specific namespace
```

## Networking in Kubernetes
Kubernetes networking allows communication between pods and services. Here are some example commands related to networking:

```bash
kubectl get pods --all-namespaces -o wide # List all pods across all namespaces with detailed information including IP addresses

```

## Service Management
Services in Kubernetes are used to expose applications running on a set of pods. Here are some example commands to create and manage services:
```bash
kubectl expose pod myapp --type=NodePort --port=8080 # Expose a pod as a service
kubectl expose deployment my-dep --type=LoadBalancer --port=80 --target-port=8080 # Expose a deployment as a service

# Delete a service
kubectl delete service my-service
``` 

## Fixing Common Issues
Here are some commands to troubleshoot and fix common issues in Kubernetes with pods and deployments:

```bash
kubectl get events --sort-by='.metadata.creationTimestamp' | less # View recent events in the cluster to identify issues    
kubectl describe pod myapp | less # Get detailed information about a specific pod to diagnose issues

docker kill $(docker ps -q -f name=k8s_mealie) # Forcefully stop a misbehaving container (use with caution)

kubectl scale deployment mealie --replicas=0 -n stevens # Scale down a deployment to zero replicas to stop all pods
kubectl delete pod myapp --force --grace-period=0 # Forcefully delete a pod that is stuck in terminating state
```


## Storage Management
Kubernetes provides various storage options for managing persistent data. Here are some example commands related to storage:

### Ephemeral Storage Example
**Ephemeral storage** is temporary storage that is tied to the lifecycle of a pod(i.e the volume is deleted when the pod is deleted). Here are some example commands and a YAML configuration for using ephemeral storage in a pod.

Types of ephemeral storage:
- emptyDir
- configMap

```bash
kubectl get pvc # List all Persistent Volume Claims (PVCs) in the current namespace

# Explain this yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec: # Define the pod specification
  containers:
  - image: registry.k8s.io/test-webserver
    name: test-container
    volumeMounts: # Mount the volume at /cache inside the container
    - mountPath: /cache
      name: cache-volume # Reference to the volume defined below
  volumes: # Define the volume to be used by the pod
  - name: cache-volume
    emptyDir: # Use an emptyDir volume type
      sizeLimit: 500Mi # Set size limit for the emptyDir volume
      medium: Memory
```

### Persistent Storage Example
**Persistent storage** is storage that persists beyond the lifecycle of a pod.

#### Persistent Volume (PV) and Persistent Volume Claim (PVC)
**Persistent Volumes (PVs)** are storage resources in a Kubernetes cluster, while **Persistent Volume Claims (PVCs)** are requests for those resources by users. Here are some example commands and YAML configurations for using PVs and PVCs in Kubernetes.

```bash