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

