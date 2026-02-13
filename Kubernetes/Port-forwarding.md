# Kubernetes Port Forwarding Runbook

This guide gives sequential steps to port-forward Kubernetes workloads across common environments: local terminal, WSL, remote SSH host, and bastion/private network access.

## 1. Prerequisites

1. Confirm `kubectl` is installed.
2. Confirm your kubeconfig context points to the intended cluster.
3. Identify the namespace and workload you want to reach.
4. Choose a free local port on the machine where the command will run.

```bash
kubectl version --client
kubectl config current-context
kubectl get ns
kubectl -n <namespace> get pods,svc,deploy
```

## 2. Core Command Patterns

Use one of these targets:

```bash
kubectl -n <namespace> port-forward pod/<pod-name> <local-port>:<container-port>
kubectl -n <namespace> port-forward svc/<service-name> <local-port>:<service-port>
kubectl -n <namespace> port-forward deploy/<deployment-name> <local-port>:<container-port>
```

Example:

```bash
kubectl -n stevens port-forward svc/mealie 9000:9000
```

## 3. Environment A: Local Linux/macOS/Windows Terminal

Use this when your browser and `kubectl` command run on the same machine.

1. Start the forward.
2. Keep that terminal open.
3. Test from the same machine.
4. Open the app in your browser.
5. Stop with `Ctrl+C` when done.

```bash
kubectl -n <namespace> port-forward svc/<service-name> 8080:80
curl -I http://127.0.0.1:8080
```

Open: `http://localhost:8080`

## 4. Environment B: WSL Terminal on Windows

Use this when `kubectl` runs inside WSL.

1. In WSL, run the port-forward command.
2. In WSL, verify the endpoint is reachable.
3. In Windows, try `http://localhost:<port>`.
4. If Windows cannot reach it, re-run with `--address 0.0.0.0` and then use the WSL IP.

```bash
# In WSL
kubectl -n <namespace> port-forward svc/<service-name> 8080:80
curl -I http://127.0.0.1:8080
ss -ltnp | rg 8080
hostname -I

# If Windows cannot access localhost mapping:
kubectl -n <namespace> port-forward --address 0.0.0.0 svc/<service-name> 8080:80
```

Fallback URL from Windows: `http://<wsl-ip>:8080` (only after `--address 0.0.0.0`)

## 5. Environment C: Command Runs on Remote Host (for example, `sochi`)

Use this when you SSH into a remote machine and run `kubectl` there.

1. SSH to the remote host.
2. On the remote host, start `kubectl port-forward`.
3. On the remote host, verify `127.0.0.1:<port>` works.
4. From your local machine, open a second terminal and create an SSH local tunnel.
5. Access `http://localhost:<port>` on your local machine.

```bash
# Terminal 1 (remote host session)
ssh endy@<remote-host>
kubectl -n <namespace> port-forward svc/<service-name> 8080:80
curl -I http://127.0.0.1:8080

# Terminal 2 (local machine)
ssh -N -L 8080:127.0.0.1:8080 endy@<remote-host>
```

Open locally: `http://localhost:8080`

## 6. Environment D: Bastion Host to Private Cluster

Use this when cluster access is only available through a bastion/jump host.

### Option 1: `kubectl` runs on bastion

1. SSH to bastion.
2. Start `kubectl port-forward` on bastion.
3. From your local machine, tunnel local port to bastion localhost port.
4. Browse locally on your machine.

```bash
# On bastion
kubectl -n <namespace> port-forward svc/<service-name> 8080:80

# On local machine
ssh -N -L 8080:127.0.0.1:8080 user@<bastion-host>
```

### Option 2: `kubectl` runs on internal host behind bastion

1. SSH to the internal admin host via bastion (`-J`).
2. Start `kubectl port-forward` there.
3. Open a second local terminal and tunnel with `-J`.
4. Browse locally.

```bash
# Terminal 1
ssh -J user@<bastion-host> user@<internal-host>
kubectl -n <namespace> port-forward svc/<service-name> 8080:80

# Terminal 2
ssh -N -J user@<bastion-host> -L 8080:127.0.0.1:8080 user@<internal-host>
```

Open locally: `http://localhost:8080`

## 7. Verification Checklist

1. Terminal shows: `Forwarding from 127.0.0.1:<port>`.
2. Local test works: `curl -I http://127.0.0.1:<port>`.
3. The correct Kubernetes namespace/context is in use.
4. All required SSH tunnels are active (if remote/bastion flow).

## 8. Troubleshooting

`error: unable to forward port because pod is not running`

```bash
kubectl -n <namespace> get pods
kubectl -n <namespace> describe pod <pod-name>
```

`address already in use`

```bash
ss -ltnp | rg <local-port>
# choose a different local port, for example 18080:80
```

`works on remote host but not on local machine`

```bash
# check local SSH tunnel command and keep it running with -N
ssh -N -L <local-port>:127.0.0.1:<local-port> user@<remote-host>
```

`forwards but app is unreachable`

```bash
# verify you used the right target port
kubectl -n <namespace> get svc <service-name> -o yaml
kubectl -n <namespace> get pod <pod-name> -o yaml
```

## 9. Security Notes

1. Default bind is localhost (`127.0.0.1`) and is safest.
2. Avoid `--address 0.0.0.0` unless you intentionally want LAN access.
3. Stop forwards and SSH tunnels when finished.

## 10. Related Guides

1. Kubernetes overview: `Kubernetes/README.md`
2. ConfigMaps runbook: `Kubernetes/configMaps/README.md`
3. Volumes and volumeMounts (beginner): `Kubernetes/volumes/README.md`
