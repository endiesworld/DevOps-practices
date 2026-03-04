# Kubernetes Secrets: Practical Guide (Including Flux + SOPS + age)

This guide covers how to create and consume Kubernetes Secrets safely, then how to store encrypted Secret manifests in Git with Flux.

## Must-Fix Issues Addressed in This Revision

1. Corrected confusion between hashing and base64 encoding.
2. Replaced claims that `data` is the only declarative option; added `stringData`.
3. Removed malformed/broken command blocks and trailing unrelated YAML.
4. Fixed inconsistent Flux API examples and incomplete Kustomization snippets.
5. Corrected `.sops.yaml` behavior explanation (encryption rules, not Flux decryption rules).
6. Corrected filename/decryption assumptions (`.enc.yaml` is convention, not hard requirement).
7. Added shell-history warning for `--from-literal` secrets.
8. Added warning that base64 is not encryption.

## 1) What a Kubernetes Secret Is

A Secret is a Kubernetes API object for storing sensitive values (passwords, API keys, tokens, private keys).

Important security reality:

- `Secret.data` values are base64 encoded, not encrypted by default.
- Base64 only transforms format; it does not provide cryptographic protection.
- For stronger protection, use etcd encryption-at-rest and strict RBAC.

## 2) Create Secrets (Imperative)

### Quick method (acceptable for testing)

```bash
kubectl create secret generic my-secret \
  --from-literal=username=myuser \
  --from-literal=password=mypassword
```

Security note:

- `--from-literal` can leak sensitive values into shell history/process args.
- Prefer `--from-file` or stdin-based workflows for production use.

### Safer file-based method

Create local files:

```bash
printf 'myuser' > username.txt
printf 'mypassword' > password.txt
```

Create Secret from files:

```bash
kubectl create secret generic my-secret \
  --from-file=username=./username.txt \
  --from-file=password=./password.txt
```

## 3) Create Secrets (Declarative YAML)

You have two valid declarative patterns.

### Option A: `stringData` (recommended for authoring)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: default
type: Opaque
stringData:
  username: myuser
  password: mypassword
```

- Kubernetes converts `stringData` into `data` on write.

### Option B: `data` (base64 values)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: default
type: Opaque
data:
  username: bXl1c2Vy
  password: bXlwYXNzd29yZA==
```

Encode/decode helpers:

```bash
printf 'myuser' | base64
printf 'mypassword' | base64
printf 'bXl1c2Vy' | base64 -d
```

Apply:

```bash
kubectl apply -f secret.yaml
```

## 4) View and Inspect Secrets

```bash
kubectl get secrets
kubectl get secret my-secret -o yaml
kubectl describe secret my-secret
```

- `describe` shows keys/metadata but not decoded values.
- `-o yaml` shows base64 in `data`.

## 5) Use Secrets in Pods

### A) As environment variables (`envFrom`)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod-envfrom
spec:
  containers:
  - name: my-container
    image: nginx
    envFrom:
    - secretRef:
        name: my-secret
```

Note:

- Each key becomes an environment variable.
- Keys that are invalid env var names can be skipped by Kubernetes.

### B) As specific environment variables (`env` + `secretKeyRef`)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod-env
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

### C) As files in a Secret volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod-volume
spec:
  containers:
  - name: my-container
    image: nginx
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secret-data
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: my-secret
```

- Each Secret key is mounted as a file under `/etc/secret-data`.

## 6) GitOps Secrets with Flux + SOPS + age

Goal:

- Store encrypted Secret manifests in Git.
- Let Flux decrypt in-cluster during reconciliation.

High-level flow:

1. Workstation encrypts manifest with SOPS using age public key.
2. Flux `kustomize-controller` uses age private key from `flux-system` Secret.
3. Flux applies decrypted manifest to target namespace.

Important:

- You do not install `sops` or `age` on Kubernetes nodes.

## 7) Install Tooling (Workstation)

Ubuntu/WSL example:

```bash
sudo apt update
sudo apt install -y age
age --version
```

Install `sops` from official release instructions.

## 8) Generate age Keys

```bash
age-keygen -o age.agekey
age-keygen -y age.agekey
```

- `age.agekey` (private): keep secure, never commit.
- `age1...` (public recipient): safe to share for encryption.

Optional helper:

```bash
export AGE_PUBLIC_KEY="$(age-keygen -y age.agekey)"
```

## 9) Store age Private Key in Cluster for Flux

Create/update decryption key Secret in `flux-system`:

```bash
kubectl create secret generic sops-age \
  --namespace flux-system \
  --from-file=age.agekey=./age.agekey \
  --dry-run=client -o yaml | kubectl apply -f -
```

Verify:

```bash
kubectl -n flux-system get secret sops-age
```

Note:

- Secret name `sops-age` is arbitrary, but must match `spec.decryption.secretRef.name` in Flux Kustomization.
- Key filename should end with `.agekey`.

## 10) Configure `.sops.yaml`

Create `.sops.yaml` at repo root:

```yaml
creation_rules:
  - path_regex: '.*\.enc\.yaml$'
    encrypted_regex: '^(data|stringData)$'
    age: age1REPLACE_WITH_YOUR_PUBLIC_KEY
```

What this does:

- Defines how `sops` encrypts files matching the path regex.
- `encrypted_regex` means only `data`/`stringData` fields are encrypted.

## 11) Encrypt a Secret Manifest

Example plaintext manifest (`test-secret.yaml`):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: test-secret
  namespace: staging
type: Opaque
stringData:
  username: myuser
  password: mypassword
```

Encrypt to a separate file:

```bash
sops --encrypt --age "$AGE_PUBLIC_KEY" \
  --encrypted-regex '^(data|stringData)$' \
  --output test-secret.enc.yaml \
  test-secret.yaml
```

Or encrypt in place:

```bash
sops --encrypt --age "$AGE_PUBLIC_KEY" \
  --encrypted-regex '^(data|stringData)$' \
  --in-place test-secret.yaml
```

Local decryption test:

```bash
SOPS_AGE_KEY_FILE=./age.agekey sops --decrypt test-secret.enc.yaml
```

## 12) Configure Flux Kustomization Decryption

Example Flux `Kustomization` (`kustomize.toolkit.fluxcd.io/v1`):

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 1m0s
  path: ./apps/staging
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

Notes:

- `decryption.secretRef.name` must match Secret created in `flux-system`.
- Decryption behavior depends on SOPS metadata + Flux decryption config, not only file extension.

## 13) Reconcile and Verify

```bash
kubectl apply -f clusters/staging/apps.yaml
flux reconcile source git flux-system -n flux-system
flux reconcile kustomization apps -n flux-system
kubectl -n staging get secret test-secret
```

Check controller logs if needed:

```bash
kubectl -n flux-system logs deploy/kustomize-controller --tail=200
```

## 14) Namespace Rule (Management vs Consumption)

- `flux-system` namespace holds decryption key Secret (for Flux internals).
- Application Secrets should exist in the target app namespace (for app consumption).

Reason:

- Workloads in one namespace should not depend on reading secrets from `flux-system`.

## 15) Common Mistakes

1. Treating base64 as encryption.
2. Committing `age.agekey` to Git.
3. Classifying `.sops.yaml` as Flux config (it is SOPS encryption config).
4. Using mismatched `secretRef.name` between Flux Kustomization and `flux-system` Secret.
5. Assuming `.enc.yaml` suffix alone controls Flux decryption.
6. Reusing identical Pod names in multiple examples and expecting all to apply cleanly.
