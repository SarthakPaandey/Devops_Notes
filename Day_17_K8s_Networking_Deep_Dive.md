# Day 17: Advanced Kubernetes Networking & Security
**Date:** 10th December

## 1. The CNI (Container Network Interface)
K8s doesn't implement networking itself. It relies on plugins.
*   **Flannel:** Simple Overlay (VXLAN). Great for learning. No Network Policies support.
*   **Calico:** BGP (Layer 3) routing. High performance. Supports Network Policies. Standard for Prod.
*   **Cilium:** eBPF-based security & observability. Very advanced.
*   **AWS VPC CNI:** Pods get real VPC IPs. High performance on AWS.

---

## 2. Network Policies (Firewalls for Pods)
By default, K8s is a flat network. Any pod can talk to any pod. This is insecure.
**NetworkPolicy** acts like a whitelist firewall.

### A. Default Deny-All (Zero Trust)
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {} # Selects all pods in namespace
  policyTypes:
  - Ingress
  - Egress
```
*   *Effect:* Blocks ALL incoming and outgoing traffic.

### B. Allow Specific Traffic
Allow `frontend` to talk to `backend` on port 80.
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80
```

---

## 3. Ingress Controllers (L7 Routing)
Service `LoadBalancer` is expensive (1 LB per service). Ingress allows 1 LB for *many* services using Host/Path routing.

### Components
1.  **Ingress Resource:** The rule (`example.com/api -> service-api`).
2.  **Ingress Controller:** The implementation (Nginx/Traefik/HAProxy). A Pod running Nginx + a Go binary that watches Ingress Resources and reloads `nginx.conf`.

### Example
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
spec:
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

---

## 4. Service Mesh (Istio / Linkerd)
When you have 100+ microservices, Network Policies manage "Access", but who manages "Traffic"?
**Service Mesh adds a Sidecar Proxy (Envoy) to every Pod.**

### Features
1.  **mTLS:** Automatic encryption between pods.
2.  **Traffic Shifting:** "Send 1% traffic to v2" (Canary).
3.  **Observability:** "Path A is slow". (Distributed Tracing).
4.  **Resilience:** Retries, Timeouts, Circuit Breakers (configured in YAML, not code).

### Sidecar Injection
Automatic.
1.  Label namespace: `kubectl label namespace default istio-injection=enabled`.
2.  Restart pods.
3.  Istio Mutating Webhook injects `istio-proxy` container into every new pod.

---

## 5. Interview Questions
1.  **Q: Difference between Ingress and Service?**
    *   *A:* **Service** creates a stable IP (L4). **Ingress** manages the external access (L7 HTTP/HTTPS) and routing based on Host/Path. Ingress *needs* a Service to route to.
2.  **Q: How does CNI work?**
    *   *A:* When Kubelet starts a pod, it calls the CNI binary (e.g., `/opt/cni/bin/calico`). The binary creates the veth pair, assigns an IP from IPAM, and sets up routes.
3.  **Q: Why use a Service Mesh?**
    *   *A:* To offload network logic (Retries, Circuit Breaking, TLS) from the application code to the infrastructure layer. "Dumb Pipes, Smart Endpoints" becomes "Smart Pipes, Dumb Endpoints".
4.  **Q: What happens if I apply a Network Policy but have no CNI plugin that supports it?**
    *   *A:* **Nothing.** The policy is accepted by API Server but ignored. Traffic flows freely. (Silent failure security risk). Always check your CNI capabilities.
