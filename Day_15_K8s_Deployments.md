# Day 15: Deep Dive into Kubernetes Deployments
**Date:** 5th December

## 1. Why use Deployments?
Deployments provide declarative updates for Pods and ReplicaSets.
*   **ReplicaSet:** Ensures N pods are running.
*   **Deployment:** Ensures N pods are running AND handles updates (Rolling Update, Rollback).
*   **Hierarchy:** Deployment -> Manages ReplicaSet -> Manages Pods.

## 2. Deployment Strategies

### A. Recreate Strategy
*   **Logic:** Kill ALL existing pods (N=0). Then create ALL new pods (N=3).
*   **Pros:** Simple. No API compatibility issues (v1 and v2 never run together).
*   **Cons:** Downtime.
*   **Use Case:** Development environments. Data migrations that lock the DB.

### B. Rolling Update (Default)
Updates pods in a rolling fashion.
*   **`maxUnavailable`:** The max number of pods that can be unavailable during the update. (Default: 25%).
*   **`maxSurge`:** The max number of pods that can be created *above* the desired number of pods. (Default: 25%).

**The Math (Replicas=4):**
1.  **Start:** 4 Old (v1).
2.  **Surge:** Create 1 New (v2). Total = 5 (4 Old, 1 New).
3.  **Unavailable:** Kill 1 Old. Total = 4 (3 Old, 1 New).
4.  **Repeat:** Until 4 New (v2) are running.

```yaml
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 1 extra pod allowed (5 total)
      maxUnavailable: 0  # 0 pods allowed to be down (Always 4 running)
```

### C. Blue/Green Deployment
*   **Logic:** Maintain two identical environments (Blue=v1, Green=v2). Switch traffic instantly via Service Selector.
*   **Pros:** Zero downtime. Instant rollback.
*   **Cons:** Double the cost (Need 2x resources).
*   **Manual Workflow:**
    1.  Deploy Blue (`app: v1`). Service selector points to `app: v1`.
    2.  Deploy Green (`app: v2`). Wait for health checks.
    3.  Update Service selector to `app: v2`.
    4.  Delete Blue if stable.

### D. Canary Deployment
*   **Logic:** Roll out v2 to a small subset of users (5%).
*   **Pros:** Risk mitigation. If v2 sucks, only 5% suffer.
*   **Tooling:** Istio / Flagger / Argo Rollouts.
*   **K8s Native (Poor Man's Canary):**
    1.  Deployment A (v1): 9 Replicas.
    2.  Deployment B (v2): 1 Replica.
    3.  Service selects `app: myapp` (Matches both).
    4.  Traffic splits 90/10.

---

## 3. Managing Updates in Practice

### Creating a Deployment
```bash
kubectl create deployment nginx --image=nginx:1.14.2 --replicas=3
kubectl get deployments
```

### Updating Image
```bash
kubectl set image deployment/nginx nginx=nginx:1.16.1 --record
# --record saves the command in history
```

### Checking Status
```bash
kubectl rollout status deployment/nginx
# "Waiting for rollout to finish: 2 out of 3 new replicas have been updated..."
```

### Pausing/Resuming
Useful for making multiple changes (Image + CPU limit) without triggering multiple rollouts.
```bash
kubectl rollout pause deployment/nginx
kubectl set image ...
kubectl set resources ...
kubectl rollout resume deployment/nginx
```

---

## 4. Rollbacks (Undo)
Something went wrong. v2 is crashing.
1.  **Check History:**
    ```bash
    kubectl rollout history deployment/nginx
    # REVISION  CHANGE-CAUSE
    # 1         kubectl create deployment...
    # 2         kubectl set image...
    ```
2.  **Undo to Previous:**
    ```bash
    kubectl rollout undo deployment/nginx
    ```
3.  **Undo to Specific Revision:**
    ```bash
    kubectl rollout undo deployment/nginx --to-revision=1
    ```

---

## 5. Interview Questions
1.  **Q: How does Deployment manage ReplicaSets?**
    *   *A:* It creates a new RS for every revision. If you update from v1 to v2, it scales up RS-v2 (0->3) and scales down RS-v1 (3->0). The old RS is kept (empty) for rollback purposes.
2.  **Q: What is `revisionHistoryLimit`?**
    *   *A:* The number of old ReplicaSets to keep for rollback. Default is 10. If you set it to 0, you cannot rollback.
3.  **Q: Can I use HPA with Deployments?**
    *   *A:* Yes. HPA scales the `replicas` field of the Deployment. (Actually, it scales the RS directly, and Deployment syncs, but user thinks in Deployments).
