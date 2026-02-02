# Securing Kubernetes Pods

This document explains how to secure Kubernetes Pods using the `securityContext` field in the Pod Manifest. The goal is to minimize the risk of container escape attacks by implementing a defense-in-depth strategy.

## The Core Problem
A container is not a Virtual Machine (VM). It is just a standard Linux process running on your physical server, but with a "mask" on.

**Shared Kernel:** If a process is root (UID 0) inside the container, it is root on the host kernel.

```bash
kubectl exec -it -n linkding pod/linkding-65d964754d-fgh8n -c linkding -- sh
# Inside the container
id
uid=0(root) gid=0(root) groups=0(root)
# On the host
id
uid=0(root) gid=0(root) groups=0(root)

cat /proc/1/status | grep '^Uid:' # Shows UIDs of host processes
Uid:    0    0    0    0

```

**The Risk:** If an attacker escapes the container (the "mask" slips), they have administrative control over your entire node.

## The Strategy: Defense in Depth
We cannot stop every attack, but we can minimize the damage. We use the **securityContext** field in the Pod Manifest to build multiple layers of defense.

### Layer 1: Identity (Who are you?)
**Goal:** Ensure the process never runs as root.

- How: Set runAsUser to a high number (like 1000).

- Why: If an attacker breaks out, they land on the host as a harmless user with no permission to change system files.

### Layer 2: Capabilities (What can you do?)
- Goal: Break "root" privileges into small slices and throw away the ones we don't need.

- The "Swiss Army Knife" Analogy: Root access is a knife with 30+ tools (mounting drives, changing clocks, binding ports). Most apps only need the "spoon" (network access).

### Best Practice:

DROP ALL capabilities first.

ADD back only what is strictly necessary (e.g., NET_BIND_SERVICE).

3. The Implementation (The YAML)
This is how we translate the strategy into code.

```YAML
apiVersion: v1
kind: Pod
metadata:
  name: secure-guard
spec:
spec:
  securityContext:
    runAsUser: 1000           # Layer 1: Be a standard user
    runAsGroup: 3000
  containers:
  - name: my-app
    image: my-app-image
    securityContext:
      allowPrivilegeEscalation: false  # Stop user from becoming root
      capabilities:
        drop: ["ALL"]         # Layer 2: Drop the Swiss Army Knife
        add: ["NET_BIND_SERVICE"] # Add back only the spoon
```