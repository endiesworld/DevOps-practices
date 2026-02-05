# Secrets Management in Kubernetes
A Secret is a Kubernetes object that allows you to store sensitive information, such as passwords, OAuth tokens, and SSH keys. Secrets help you manage sensitive data separately from application code, enhancing security and flexibility. When a pod is created, inject the secret into the pod so the sensitive data is available as environment variables or as files in a volume.

There are two ways to create and use Secrets in Kubernetes:
1. Imperative way using `kubectl` commands
```bash
kubectl create secret generic my-secret --from-literal=username=myuser --from-literal=password=mypassword
```
**Explanation:** This command creates a Secret named `my-secret` with two key-value pairs: `username=myuser` and `password=mypassword`.
If you wish to add additional key value pairs, simply specify the from literal options multiple times.

2. Declarative way using a Secret definition file (YAML or JSON). So while creating a secret with a declarative approach, you must specify the secret values in a hashed format.

So you must specify the data in an encoded form like this.
Create a file named `secret.yaml` with the following content:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  username: bXl1c2Vy # base64 for "myuser"
  password: bXlwYXNzd29yZA== # base64 for "mypassword"
```
**Encoding Note:** Convert your plain text values to base64 encoded format using the following command:
```bash
echo -n 'myuser' | base64 # 'myuser' becomes bXl1c2Vy
echo -n 'mypassword' | base64 # 'mypassword' becomes bXlwYXNzd29yZA==
``` 

**Decoding Note:** To decode a base64 encoded value, you can use:
```bash
echo 'bXl1c2Vy' | base64 --decode
```

Then apply the file using kubectl:
```bash
kubectl apply -f secret.yaml
```
## Viewing Secrets
To view the created Secrets, use the following command:
```bash
kubectl get secrets
kubectl describe secret my-secret    
```
## Using a Secret in a Pod
You can use a Secret in a Pod in two ways:
1. As environment variables
2. As files in a volume
### 1. Using Secret as Environment Variables
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
    - secretRef:
        name: my-secret # Reference the Secret here
```

**Note:** The above example mounts all key-value pairs from the Secret as environment variables in the container. Each key in the Secret becomes an environment variable with the corresponding value.

To inject specific keys from the Secret as environment variables, you can use the `env` field like this:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: nginx
    env:
    - name: MY_USERNAME
      valueFrom:
        secretKeyRef:
          name: my-secret
          key: username
    - name: MY_PASSWORD
      valueFrom:
        secretKeyRef:
          name: my-secret
          key: password
```

Then create the Pod using:
```bash
kubectl apply -f pod-env.yaml
```
### 2. Using Secret as Files in a Volume
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
    - name: secret-volume
      mountPath: /etc/secret-data
  volumes:
  - name: secret-volume
    secret:
      secretName: my-secret
```
Then create the Pod using:
```bash
kubectl apply -f pod-volume.yaml
```

**Note:** In this example, the Secret `my-secret` is mounted as files in the `/etc/secret-data` directory inside the container. Each key in the Secret becomes a file with the corresponding value as its content.



## GitOps Secrets with Flux + SOPS + age (Encrypt in Git, Decrypt in Cluster)

This README documents a clean workflow to **store Kubernetes `Secret` manifests encrypted in Git** and have **Flux decrypt them inside the cluster** during reconciliation.

## How it works (high level)

- **You (workstation / WSL):**
  - install `age` + `sops`
  - generate an **age keypair**
  - encrypt `Secret` YAML using **SOPS** + **age public key**
  - commit/push encrypted YAML to Git

- **Cluster (Flux `kustomize-controller`):**
  - pulls the repo
  - uses the **age private key** stored as a **Kubernetes Secret**
  - decrypts SOPS files *in-cluster*
  - applies the decrypted resources to the API server

> Important: You do **NOT** install `sops` or `age` on Kubernetes nodes. Decryption happens inside Flux controller pods.

---

## Prerequisites

- Flux v2 installed and bootstrapped
- `kubectl` access to the cluster
- A GitOps repo structure with Flux `Kustomization` resources (e.g., `clusters/staging/...`)
- Workstation tools:
  - `age`
  - `sops`

---

## Install tooling (Ubuntu / WSL)

### Install `age` (Ubuntu/WSL)
```bash
sudo apt update
sudo apt install -y age
age --version
```

### Install `sops` (Ubuntu/WSL)

use the [official instructions](https://github.com/getsops/sops/releases)

### Generate age keys (one-time per environment)
Generate a private key file:
```bash
age-keygen -o age.agekey
Print the public key (recipient):
age-keygen -y age.agekey
# outputs: age1...
```
The public key is used to encrypt files. The private key is used to decrypt files.
What to keep where

Private key (age.agekey): keep safe (NEVER commit to Git)

Public key (age1...): safe to share/use for encryption

### Store the age private key in the cluster (one-time)

Create a Kubernetes Secret in flux-system that contains your private key.

The key inside the Secret must end with .agekey.
```bash
kubectl -n flux-system create secret generic sops-age \
  --from-file=identity.agekey=age.agekey
# Verify:

kubectl -n flux-system get secret sops-age
```

### Configure Flux to use the age private key
Edit your `Kustomization` resource (e.g., `clusters/staging/kustomization.yaml`) to add the decryption configuration:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
    name: my-app
    namespace: flux-system
spec:
    interval: 10m0s
    decryption:
        provider: sops
        secretRef:
            name: sops-age

```
### Encrypt a Secret manifest using SOPS + age
Create a Kubernetes Secret manifest (e.g., `secret.yaml`):
```yaml
apiVersion: v1
kind: Secret
metadata:
    name: my-secret
type: Opaque
data:
    username: bXl1c2Vy # base64 for "myuser"
    password: bXlwYXNzd29yZA== # base64 for "mypassword"
```
Encrypt it using SOPS and the age public key:
```bash
sops --encrypt --age <age-public-key> --output secret.enc.yaml secret.yaml
```
Commit and push the encrypted file (`secret.enc.yaml`) to your Git repo.

### Apply changes
Flux will automatically pull the changes, decrypt the Secret using the age private key stored in the cluster, and apply it to the Kubernetes API server.
    image: my-app-image:latest
    securityContext:
      capabilities:
        drop: ["ALL"]         # Layer 2: Drop the Swiss Army Knife
        add: ["NET_BIND_SERVICE"] # Add back only the spoon
``` 
