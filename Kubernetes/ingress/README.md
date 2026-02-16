# Ingress

Ingress is a Kubernetes API object (a resource) just like Pod/Service/Deployment.
**More precisely:**
- It’s a namespaced resource (lives inside a namespace).
- It’s part of the networking.k8s.io API group.
- You create it with YAML and kubectl apply, and it’s stored in etcd like other objects.

**But the key nuance:**
- Ingress = desired routing rules (declarative config)
- Ingress Controller = the thing that makes those rules real (by running a reverse proxy and programming it)

To really understand what Ingress is, it helps to first understand the problems it solves. Before Ingress existed, Kubernetes had two primary ways to expose applications, and both had limitations for large scale use:

- NodePort: This opens a specific port (like 30000) on every server (node) in your cluster. It's simple, but you end up with ugly URLs like http://my-node-ip:30000.

- LoadBalancer: This asks your cloud provider (like AWS or Google Cloud) to give you a dedicated public IP address for that specific service.

**The "One-to-One" Problem**
Imagine you have 10 different websites running in your cluster.
- If you use the LoadBalancer method, you have to pay for 10 distinct static IP addresses and 10 cloud load balancers.
- If you decide to use Nodeport, It creates a usability problem. NodePort operates at Layer 4 (TCP/UDP), not Layer 7 (HTTP). It cannot look at the URL (foo.com vs bar.com).
To host 10 services, you would need to open 10 different ports (e.g., :30001, :30002, :30003). Your users would have to type http://your-ip:30001 to get to App A and http://your-ip:30002 to get to App B.

- The Cloudflare Tunnel Solution (Advanced)
Using cloudflared (Cloudflare Tunnel) essentially bypasses the traditional Kubernetes Ingress model.
<How it works:> Instead of opening a door in (Ingress), you run a pod that creates a secure tunnel out to Cloudflare's edge network. Does it solve the problem? Yes. It allows you to host multiple services on standard ports (80/443) without paying for a cloud LoadBalancer or static IPs. Cloudflare handles the routing based on the hostname.

Ingress also solves this by sitting in front of all of them. It uses just one IP address and one load balancer to route traffic to all 10 websites based on the URL.

## Cloudflared Tunnel Deployment vs. Ingress (Where Ingress fits in your setup (Cloudflared + K8s))

**NOTE:** There is a nuace here, Kubernetes ingress can exist and function on its own without Cloudflared, Cloudflared is only needed here if you want internet access and not limiting ingress to local access withing your network.

+ Pattern 1: Tunnel → Service (no Ingress)
-  Cloudflare hostname/path routes straight to a Kubernetes Service
**flow:**
- Internet → Cloudflare → Tunnel → cloudflared Pod → Service → App Pods
Simple, but you end up defining routing mostly in Cloudflare (per-service/per-path).

+ Pattern 2: Tunnel → Ingress Controller → Ingress rules → Services
- Cloudflare tunnel forwards to one internal endpoint (the Ingress controller Service).
Then Kubernetes Ingress objects do the host/path routing inside the cluster.
**Flow:**
Internet → Cloudflare → Tunnel → cloudflared Pod → Ingress Controller → (Ingress rules) → Services → Pods
This is the “Ingress becomes your internal traffic router” approach.

In standard Kubernetes, we split this networking responsibility with ingress into two parts to keep things flexible:
- The Ingress Resource: A YAML file where you write the rules (e.g., "Send api.com to the api-service").
- The Ingress Controller: The actual software (like Nginx, Traefik, or even Cloudflare's agent) that reads those rules and moves the traffic.
```sh
# To see if you have an ingress controller
kubectl get ingressclass
```

## What “Ingress” is (concept)

Ingress is a Kubernetes way to define HTTP/HTTPS routing rules like:
- requests to example.com go to Service A
- requests to example.com/api go to Service B
- enable TLS (HTTPS) for a hostname

Ingress is not the thing that actually routes traffic by itself. It’s just a set of rules.
**What actually does the routing**
An Ingress Controller (like NGINX Ingress, Traefik, HAProxy, etc.) is the running software in your cluster that:
- watches Ingress objects
- configures a real reverse proxy / load balancer
- forwards traffic to your Services

**Mental model:** 
Ingress (rules) + Ingress Controller (engine) = working HTTP entry to cluster