```bash
kubectl run -h | less # Display help information for the 'kubectl run' command "/dry-run" option

kubectl run myapp --image=myimage --dry-run=client -o yaml > myapp-deployment.yaml

kubectl apply -f myapp-deployment.yaml # Apply the generated YAML file to create the deployment. There is also create option.

kubectl get pods # Verify that the pod is created

kubectl describe pod myapp | less # Get detailed information about the created pod. Use the JKL keys to navigate up and down.
```

