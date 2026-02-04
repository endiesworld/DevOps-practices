# GitOps Secrets with Flux + SOPS + age (Encrypt in Git, Decrypt in Cluster)

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
