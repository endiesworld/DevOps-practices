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

