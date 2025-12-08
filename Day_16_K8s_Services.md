# Day 16: Kubernetes Services & Networking
**Date:** 8th December

## 1. The Dynamic Nature of Pods
Pods are mortal. They are created and destroyed.
*   **IP Churn:** Every time a pod restarts, it gets a new IP.
*   **Problem:** If the Backend Pod IP changes, the Frontend breaks.
*   **Solution:** **Service Abstraction**. A Service provides a stable Virtual IP (VIP) and DNS name.

## 2. Types of Services Deep Dive

### A. ClusterIP (Default)
*   **Internal Only:** Exposes the Service on a cluster-internal IP.
*   **Accessible:** Only from *within* the cluster.
*   **Use Case:** Database, Redis, Internal Backend API.
*   **Traffic Flow:** `Pod A -> Service VIP (10.96.0.1) -> IPTables -> Pod B`.

### B. NodePort
*   **External Access:** Exposes the Service on each Node's IP at a static port.
*   **Range:** 30000-32767.
*   **Accessible:** `curl <NodeIP>:<NodePort>`.
*   **Use Case:** Quick debugging, Development environment.
*   **Drawback:** Only 1 service per port. If Node IP changes, clients break. Not for Prod.
*   **Traffic Flow:** `Client -> Node IP:30007 -> Service VIP -> Pod B`.

### C. LoadBalancer
*   **Cloud Native:** Provisions an external Load Balancer (AWS ELB, GCP LB).
*   **Accessible:** Public Internet.
*   **Use Case:** Production Frontend.
*   **Traffic Flow:** `Client -> AWS ELB -> NodePort -> Service VIP -> Pod B`.

### D. ExternalName
*   **DNS Only:** Maps the Service to a CNAME (e.g., `db.prod.aws.com`).
*   **Use Case:** Allow internal apps to talk to external DBs using internal DNS names. No proxying involved.

---

## 3. Service Discovery Internals

### DNS Records (CoreDNS)
K8s runs a DNS server. Every Service gets a DNS entry.
*   **Format:** `my-svc.my-namespace.svc.cluster.local`.
*   **A Record:** Maps Service Name -> Service VIP.
    *   `nslookup my-svc` -> `10.96.0.5`.
*   **SRV Record:** Maps Service Name -> Port.
    *   `_http._tcp.my-svc.my-namespace.svc.cluster.local`.

### Environment Variables
Old school discovery.
*   When a Pod starts, Kubelet injects vars for existing services.
*   `MY_SVC_SERVICE_HOST=10.96.0.5`
*   `MY_SVC_SERVICE_PORT=80`
*   **Drawback:** Only works if Service exists *before* Pod starts. Rely on DNS instead.

---

## 4. Advanced: Headless Services
Sometimes you don't want load balancing. You want to talk to a specific Pod (e.g., StatefulSets like Kafka/Zookeeper/Cassandra).
*   **Definition:** Set `clusterIP: None` in YAML.
*   **DNS Behavior:** Instead of returning 1 VIP, DNS returns **Multiple A Records** (IPs of all 3 pods). The Client decides which one to call.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-headless
spec:
  clusterIP: None # <--- Magic
  selector:
    app: database
  ports:
  - port: 80
```

---

## 5. Endpoints Object
A Service is just an abstraction. The actual list of IPs is stored in an object called **Endpoints**.
*   **Controller:** The Endpoint Controller watches Pods. If a Pod matches the Service Selector, its IP is added to the Endpoints list.
*   **Manual Inspection:**
    ```bash
    kubectl get endpoints my-svc
    # NAME      ENDPOINTS
    # my-svc    10.244.1.5:80,10.244.2.3:80
    ```
*   **Debugging:** If Service is not working, check Endpoints. If empty, your Selector is wrong.

---

## 6. Session Affinity (Sticky Sessions)
By default, traffic is Round Robin.
*   **ClientIP:** Route traffic from the same Client IP to the same Pod.
    ```yaml
    sessionAffinity: ClientIP
    sessionAffinityConfig:
      clientIP:
        timeoutSeconds: 10800
    ```

---

## 7. Interview Questions
1.  **Q: Start a pod and curl a service. It times out.**
    *   *A:* Check Network Policies (Is traffic allowed?). Check if CoreDNS is running (`kubectl get pods -n kube-system`). Check if Service Selector matches Pod Labels (`kubectl get endpoints`).
2.  **Q: What is `externalTrafficPolicy: Local`?**
    *   *A:* Preserves the Client Source IP. By default (`Cluster`), NodePort traffic might jump to another node, losing the source IP (SNAT). `Local` forces traffic to stay on the node where it hit, preserving IP, but risking imbalance.
3.  **Q: Can I point a Service to an external IP (outside cluster)?**
    *   *A:* Yes. Create a Service without a Selector. Then manually create an Endpoints object with the external IP.
4.  **Q: How does `kube-proxy` prioritize traffic?**
    *   *A:* It uses IPTables (random probability) or IPVS (Least Connection, Round Robin, etc.). IPVS is better for large scale (>1000 services).
