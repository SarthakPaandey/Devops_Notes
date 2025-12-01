# Day 13: Kubernetes Pods Deep Dive
**Date:** 1st December

## 1. Install Kubernetes using Kubeadm
For production environments, minikube is not enough. You need a multi-node cluster.
1.  **Prerequisites:** 2 GB RAM, 2 CPUs per machine. Swap disabled.
2.  **Container Runtime:** Install `containerd`.
3.  **Kube Tools:** Install `kubeadm`, `kubelet`, `kubectl`.
4.  **Initialize Master:** `kubeadm init --pod-network-cidr=10.244.0.0/16`.
5.  **Networking:** Install CNI (Calico or Flannel). `kubectl apply -f calico.yaml`.
6.  **Join Worker:** Run the `kubeadm join` command output from step 4 on worker nodes.

---

## 2. Pods: The Atomic Unit
A Pod is the smallest deployable unit in K8s. It encapsulates one or more containers.
*   **Shared Context:** Containers in a Pod share:
    *   **Network:** Same IP address. They talk via `localhost`.
    *   **Storage:** Shared Volumes.
    *   **IPC:** Shared Inter-Process Communication namespace.
*   **Analogy:** Pod = Generic Logical Host (Virtual Machine); Container = Application running in it.

## 3. Multi-Container Patterns
Why run multiple containers in one pod? Tightly coupled processes.

### Sidecar Pattern
Enhances the main container.
*   **Example:** Main: Web Server (Nginx). Sidecar: Log Shipper (Fluentd) reading logs from shared volume and pushing to S3.

### Adapter Pattern
Standardizes output.
*   **Example:** Main: Metric producer (Proprietary format). Adapter: Converts to Prometheus format.

### Ambassador Pattern
Proxy for external world.
*   **Example:** Main: App connecting to generic "localhost:5432". Ambassador: Proxies connection to a sharded DB cluster.

---

## 4. Pod Lifecycle Phases
1.  **Pending:** Accepted by API Server, but not scheduled to a node yet (downloading images).
2.  **Running:** Bound to a node, at least one container is running.
3.  **Succeeded:** All containers terminated with exit code 0.
4.  **Failed:** At least one container terminated with non-zero exit code.
5.  **Unknown:** Master lost contact with Node (Network partition).

---

## 5. Init Containers
Containers that run *before* the main app containers.
*   **Use Case:**
    *   Wait for DB to be ready (`until nc -z db 5432; do sleep 2; done;`).
    *   Seed database schema.
    *   Register service with discovery.
*   **Behavior:** Must complete successfully before main containers start.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice; sleep 2; done"]
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
```

---

## 6. Health Checks (Probes)
K8s needs to know if your app is healthy.
1.  **Liveness Probe:** "Is the app dead?"
    *   **Action:** If failed, K8s restarts the container.
    *   **Use Case:** Deadlocks, crashed logic.
2.  **Readiness Probe:** "Is the app ready to serve traffic?"
    *   **Action:** If failed, K8s removes IP from Service Endpoints (Stops sending traffic).
    *   **Use Case:** App is starting up, loading cache, warming up DB connection.
3.  **Startup Probe:** "Has the app started?"
    *   **Action:** Disables other probes until this passes.
    *   **Use Case:** Slow legacy apps (take 5 mins to boot).

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 3
  periodSeconds: 3
```

---

## 7. Interview Questions
1.  **Q: What is the difference between `Liveness` and `Readiness`?**
    *   *A:* **Liveness** restarts the container (Fixes crashes). **Readiness** stops traffic (Fixes overload/startup). If Liveness fails, the app dies. If Readiness fails, the app stays alive but invisible to users.
2.  **Q: What is the `CrashLoopBackOff` status?**
    *   *A:* It means the container starts, crashes immediately, restarts, crashes again. K8s increases the restart delay exponentially (10s, 20s, 40s...). Common causes: Missing env var, wrong CMD, OOM Kill.
3.  **Q: Can I mount a volume to multiple containers in the same Pod?**
    *   *A:* **Yes.** This is the primary way sidecars work (e.g., Main app writes logs to `/var/log`, Sidecar reads `/var/log`).
