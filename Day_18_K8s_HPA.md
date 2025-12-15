# Day 18: Advanced Scaling - HPA, VPA, and Cluster Autoscaler
**Date:** 15th December

## 1. Introduction to Auto Scaling
Manual scaling (`kubectl scale`) is reactive and slow. Auto-scaling is proactive.
**Types:**
1.  **Horizontal (HPA):** Add more pods.
2.  **Vertical (VPA):** Add more CPU/RAM to existing pods.
3.  **Cluster (CA):** Add more nodes to the cluster.

---

## 2. Horizontal Pod Autoscaler (HPA) Deep Dive
HPA automatically scales the number of Pods in a Deployment based on observed CPU utilization.

### Prerequisites (Metrics Server)
You must install `metrics-server` addon.
*   `kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml`
*   Verify: `kubectl top nodes` / `kubectl top pods`.

### How it Works (The Algorithm)
Sync Period: Every 15 seconds.
Formula:
`TargetReplicas = ceil[CurrentReplicas * ( CurrentMetricValue / DesiredMetricValue )]`

**Example:**
*   Current Replicas: 2
*   Current CPU: 100m (Total 200m)
*   Desired CPU: 50m
*   `Target = ceil[2 * (100 / 50)] = 4`.
*   **Result:** Scale from 2 to 4.

### Scaling Behavior
*   **Scale Up:** Fast. Reacts immediately into spikes.
*   **Scale Down:** Slow (5 minute stabilization window). Prevents "Thrashing" (rapidly adding/removing pods).

### Custom Metrics (Prometheus Adapter)
CPU is not enough. You want to scale on "Messages in Queue" or "Request Latency".
1.  **Install Prometheus:** Gather app metrics.
2.  **Install Prometheus Adapter:** Translates PromQL query to k8s Custom Metrics API (`custom.metrics.k8s.io`).
3.  **HPA Config:**
    ```yaml
    type: Object
    object:
      metric:
        name: requests-per-second
      target:
        type: Value
        value: 10k
    ```

---

## 3. Vertical Pod Autoscaler (VPA)
"Right-sizing" your pods.
*   **Problem:** Developers guess CPU requests (`cpu: 200m`). They are usually wrong.
*   **Solution:** VPA watches usage history and recommends values.
*   **Modes:**
    *   **Off:** Recommendations only.
    *   **Initial:** Apply recommendation only at creation time.
    *   **Auto:** Restart pods to apply new limits (Disruptive!).
*   **Conflict:** **Do NOT run HPA and VPA on CPU/Memory together.** They will fight. Use VPA for stateful apps (DBs), HPA for stateless apps.

---

## 4. Cluster Autoscaler (CA)
HPA adds pods. What if Nodes are full? Pods go into `Pending`.
**Cluster Autoscaler** watches for `Pending` pods.
*   **Action:** Talks to Cloud Provider (AWS ASG). Spins up a new EC2 instance. Join as Node.
*   **Scale Down:** If a node is underutilized (<50%) for 10 mins, drain it and terminate instance.

---

## 5. Goldilocks Protocol
Finding the "Just Right" limits.
*   **Tool:** Goldilocks (by Fairwinds).
*   **Function:** Runs VPA in "Recommendation Mode" for all pods and gives you a dashboard suggesting optimal requests/limits.

---

## 6. Interview Questions
1.  **Q: Why are my HPA pods not scaling down?**
    *   *A:* Check the stabilization window (default 5 mins). Also, check if you have a `minReplicas` set. Check if metrics are actually below target.
2.  **Q: Can HPA scale to zero?**
    *   *A:* **No.** Standard HPA minimum is 1. To scale to zero (Serverless), use **KEDA** (Kubernetes Event Driven Autoscaling).
3.  **Q: How does HPA handle missing metrics?**
    *   *A:* It assumes 100% utilization (worst case) for missing pods to prevent scaling down prematurely.
4.  **Q: Describe the interplay between HPA and CA.**
    *   *A:* HPA adds Pods -> Pods Pending (Resource Constraint) -> CA notices Pending Pods -> CA adds Node -> Pods Scheduled.
    *   HPA removes Pods -> Node becomes empty -> CA removes Node.
