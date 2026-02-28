# Kubernetes Ingress: Zero-to-Expert Guide

This guide explains Kubernetes Ingress from first principles to production operations.

## What You Will Learn

1. Why Ingress exists and what problem it solves.
2. How Ingress, Ingress Controller, Service, DNS, and TLS fit together.
3. How to deploy and test a working Ingress.
4. How to run Ingress safely in production.
5. When to use Gateway API instead of Ingress.

## Learning Path

1. Beginner: Understand traffic flow and deploy one app behind one host.
2. Intermediate: Add TLS, multiple hosts, and path-based routing.
3. Advanced: Multi-tenant patterns, security hardening, scaling, and incident troubleshooting.

## 1) Why Ingress Exists

Before Ingress, common exposure methods were:

- `NodePort`: exposes each Service on a high port on every node.
- `LoadBalancer`: creates one external load balancer per Service (in most cloud setups).

Problems at scale:

- You may end up with many load balancers and higher cost.
- Users must remember non-standard ports with `NodePort`.
- Layer-7 routing (host/path-based) is limited without an HTTP reverse proxy.

Ingress solves this by centralizing HTTP(S) routing rules so one entry point can route to many Services.

![Ingress request flow](images/ingress-request-flow.svg)

## 2) Core Concepts (Critical)

- `Ingress` (Kubernetes resource): declarative HTTP/HTTPS routing rules.
- `Ingress Controller` (running software): implements those rules using a reverse proxy.
- `IngressClass`: binds Ingress objects to a specific controller.
- `Service`: stable virtual endpoint in front of Pods.
- `DNS`: points your hostname to the controller entry point.
- `TLS Secret`: certificate/private key used for HTTPS termination.

Important distinction:

- Ingress by itself does nothing.
- A controller must be installed and reachable for traffic to flow.
- The current stable API is `networking.k8s.io/v1`.

## 3) Correcting Common Misunderstandings

These are frequent mistakes that cause confusion:

1. "Ingress is the traffic router."
   Reality: Ingress is only config. The controller is the router.

2. "Cloudflare Tunnel is required for Ingress."
   Reality: Not required. Ingress works with cloud load balancers, NodePort, bare metal LB solutions, or tunnels.

3. "Cloudflared is an Ingress Controller."
   Reality: `cloudflared` is a tunnel client, not a Kubernetes Ingress Controller.

4. "One Ingress always means one external IP for all apps forever."
   Reality: Common pattern is one controller endpoint for many apps, but architecture can vary (multiple controllers/classes).

5. "Ingress is for all protocols."
   Reality: Standard Ingress API is for HTTP/HTTPS routing. Raw TCP/UDP support is controller-specific.

## 4) Prerequisites

- A Kubernetes cluster (local or cloud).
- `kubectl` configured to the cluster.
- An installed Ingress controller (for example, NGINX Ingress).

Check controller-class availability:

```bash
kubectl get ingressclass
```

If this returns nothing, install a controller first.
If multiple classes exist, always set `spec.ingressClassName` explicitly.

## 5) Beginner Quickstart (Working End-to-End)

### Step A: Deploy app and Service

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-web
  namespace: ingress-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-web
  template:
    metadata:
      labels:
        app: hello-web
    spec:
      containers:
      - name: hello-web
        image: nginxdemos/hello:plain-text
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: hello-web
  namespace: ingress-demo
spec:
  selector:
    app: hello-web
  ports:
  - name: http
    port: 80
    targetPort: 80
```

### Step B: Create Ingress rule

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-ingress
  namespace: ingress-demo
spec:
  ingressClassName: nginx
  rules:
  - host: hello.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-web
            port:
              number: 80
```

If your controller uses a different class name, replace `nginx` with the value from `kubectl get ingressclass`.

Apply both manifests:

```bash
kubectl apply -f app-and-service.yaml
kubectl apply -f ingress.yaml
```

### Step C: Discover the Ingress entry point

Common options:

- Cloud: external IP/hostname from the controller Service.
- Minikube: run `minikube tunnel` in a separate shell for `LoadBalancer` IP assignment.
- Bare metal: depends on MetalLB, NodePort, or another edge setup.

Example lookup for NGINX Ingress controller:

```bash
kubectl -n ingress-nginx get svc ingress-nginx-controller
```

Validate object state:

```bash
kubectl get ingress -n ingress-demo
kubectl describe ingress hello-ingress -n ingress-demo
```

### Step D: Test routing locally

Option 1: Direct host header test:

```bash
curl -H "Host: hello.local" http://<INGRESS_IP_OR_DNS>/
```

Option 2: Add local hosts entry for browser testing:

```text
<INGRESS_IP_OR_DNS> hello.local
```

Then browse to `http://hello.local`.

## 6) Path Routing and Multi-App Setup

One Ingress can route to multiple Services.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-app
  namespace: ingress-demo
spec:
  ingressClassName: nginx
  rules:
  - host: apps.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

`pathType` behavior:

- `Exact`: strict match.
- `Prefix`: path prefix match.
- `ImplementationSpecific`: controller-defined behavior.

## 7) TLS (HTTPS) Done Right

Ingress TLS requires:

1. A valid certificate.
2. A Kubernetes TLS Secret in the same namespace as the Ingress.
3. DNS pointing to the Ingress endpoint.

Example TLS block:

```yaml
spec:
  tls:
  - hosts:
    - apps.example.com
    secretName: apps-example-com-tls
  rules:
  - host: apps.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

For production, use `cert-manager` with ACME (for example, Let's Encrypt) to automate issuance/renewal.

## 8) Cloudflare Tunnel vs Ingress (How They Combine)

Pattern A: Tunnel directly to Service (no Ingress)

- Flow: Internet -> Cloudflare -> cloudflared -> Service -> Pods
- Routing logic mostly managed at Cloudflare side.

Pattern B: Tunnel to Ingress Controller (recommended for Kubernetes-native routing)

- Flow: Internet -> Cloudflare -> cloudflared -> Ingress Controller -> Services -> Pods
- Kubernetes owns host/path rules via Ingress objects.

![Cloudflare patterns](images/cloudflare-ingress-patterns.svg)

## 9) Exposure Patterns Comparison

![Exposure comparison](images/exposure-patterns.svg)

Use this rule of thumb:

- `NodePort`: simple labs, low-level debugging, non-user-facing quick tests.
- `LoadBalancer`: single app/public entry where advanced routing is not needed.
- `Ingress`: many HTTP apps, centralized TLS, shared edge policy.

## 10) Production-Grade Practices

1. Run at least 2 controller replicas.
2. Use PodDisruptionBudgets and anti-affinity.
3. Restrict risky annotations (or document approved set).
4. Set request-size and timeout limits deliberately.
5. Add access logs and metrics to your observability stack.
6. Protect endpoints with WAF/rate limiting at edge or controller level.
7. Keep app-level authn/authz in the application or dedicated auth proxy.
8. Separate public and internal traffic with different ingress classes/controllers when needed.

## 11) Security Checklist

- Prefer HTTPS-only and redirect HTTP to HTTPS.
- Rotate certificates automatically.
- Keep controller image updated.
- Limit namespace RBAC for who can create/modify Ingress.
- Audit wildcard host usage.
- Use NetworkPolicies to constrain east-west traffic.

## 12) Troubleshooting Runbook

Start with these commands:

```bash
kubectl get ingress -A
kubectl describe ingress -n <namespace> <ingress-name>
kubectl get svc -A | grep -i ingress
kubectl get pods -n <controller-namespace>
kubectl logs -n <controller-namespace> <controller-pod-name> --tail=200
kubectl get endpoints -n <namespace> <service-name>
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

Common issues:

1. `404 from controller`: host/path did not match any rule.
2. `503 upstream`: Service has no healthy endpoints.
3. TLS warning: wrong cert, wrong SNI host, or secret missing/expired.
4. Ingress ignored: wrong `ingressClassName` or controller not watching that class.
5. Works in cluster, fails externally: DNS or load balancer routing problem.

## 13) Ingress vs Gateway API

Ingress remains widely used and stable for many workloads.
For new complex traffic management (advanced policy, multi-team delegation, richer route types), evaluate Gateway API.

Pragmatic recommendation:

- If your needs are host/path routing plus TLS, Ingress is usually enough.
- If you need richer traffic policy and cleaner role separation, start piloting Gateway API.

## 14) Technical Debt Removed in This Revision

This document was restructured to remove technical debt from the previous version:

1. Fixed ambiguity between Ingress resource and controller runtime.
2. Corrected Cloudflare Tunnel assumptions (optional integration, not mandatory).
3. Clarified protocol scope (HTTP/HTTPS by default in standard Ingress).
4. Added missing operational sections: TLS, debugging, production hardening, and security.
5. Added diagrams and progressive learning flow so beginners can build mental models quickly.

## 15) Next Steps for Mastery

1. Deploy two controllers with different ingress classes (`public`, `internal`).
2. Integrate `cert-manager` and verify auto-renewal behavior.
3. Load test and tune controller limits/timeouts.
4. Pilot one service with Gateway API and compare complexity.
