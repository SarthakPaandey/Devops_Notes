# Day 14: ReplicaSets & Controller Manager Logic
**Date:** 3rd December

## 1. What are ReplicaSets?
A ReplicaSet (RS) ensures that a specified number of hybrid Pod replicas are running at any given time.
*   **Goal:** Reliability. If a Pod crashes, RS creates a new one to replace it.
*   **Relationship:** It is the next-gen "ReplicationController" (RC). RC only supported Equality-based selectors (`env=prod`). RS supports Set-based selectors (`env in (prod, qa)`).
*   **Direct Usage:** Rarely used directly. Usually managed by a **Deployment**.

## 2. Labels & Selectors Deep Dive
Labels are key-value pairs attached to objects (pods). Selectors are used by Controllers to identify objects.

### Equality-Based Selectors
*   `environment = production`
*   `tier != frontend`

### Set-Based Selectors (New in RS)
*   `environment in (production, qa)`
*   `tier notin (frontend, backend)`
*   `!partition` (Label key does not exist)

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  labels:
    app: guestbook
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
    matchExpressions:
      - {key: tier, operator: In, values: [frontend]}
      - {key: environment, operator: NotIn, values: [dev]}
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google_samples/gb-frontend:v3
```

---

## 3. How Controller Manager Works (The Loop)
It uses a reconciliation loop:
1.  **Watch:** The Controller watches the API Server for changes to Pods (Create/Update/Delete).
2.  **Compare:**
    *   **Desired State:** "I want 3 pods with label `tier=frontend`".
    *   **Current State:** "I count 2 pods with label `tier=frontend`".
3.  **Act:** Create 1 new Pod using the `template` in the manifest.

### Adoption & Orphaning
*   **Adoption:** If you create a standalone Pod with label `tier=frontend`, the ReplicaSet will "adopt" it (count it towards the replicas). If the count exceeds 3, it might terminate it.
*   **Orphaning:** If you delete a ReplicaSet with `--cascade=orphan`, the Pods are left running. Useful for debugging or moving ownership.

---

## 4. Scaling
### Manual Scaling
```bash
kubectl scale rs frontend --replicas=5
```

### Auto Scaling (HPA)
The HPA controller modifies the RS `replicas` field dynamically based on CPU/Memory metrics.
```bash
kubectl autoscale rs frontend --cpu-percent=50 --min=1 --max=10
```

---

## 5. Garbage Collection
When you delete an object (Deployment/ReplicaSet), what happens to its dependents (Pods)?
1.  **Foreground Cascading Deletion:** The API Server marks the object as "Deleting". It waits for all heavy dependents (Pods) to be deleted first. Block until done.
2.  **Background Cascading Deletion:** The API Server deletes the owner object immediately. The Garbage Collector controller deletes dependents in the background. (Faster).
3.  **Orphan Deletion:** The dependents are left running (orphaned).

---

## 6. Interview Questions
1.  **Q: Difference between ReplicationController and ReplicaSet?**
    *   *A:* **Use RS.** RS supports set-based selectors (`tier in (frontend, backend)`). RC only supports equality (`tier=frontend`). Deployments manage RS, not RC.
2.  **Q: If I delete a Pod managed by an RS, what happens?**
    *   *A:* The Pod goes into `Terminating` state. The RS watcher loop notices the Current Count (2) < Desired Count (3). It immediately creates a new Pod (Pending -> Running).
3.  **Q: Can a Pod belong to two ReplicaSets?**
    *   *A:* Technically yes, if labels overlap. But the Controllers will fight ("Configuration Drift"). One RS will try to kill it (Count > Desired), the other will try to keep it. **Avoid overlapping selectors.**
