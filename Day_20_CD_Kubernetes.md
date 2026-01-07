# Day 20: Continuous Deployment to K8s & GitOps
**Date:** 7th January

## 1. Push-Based vs Pull-Based CD
*   **Push-Based (Legacy):** CI Pipeline (Jenkins/GitHub Actions) runs `kubectl apply`.
    *   **Pros:** Easy to set up.
    *   **Cons:** Pipeline needs admin access to Cluster (Security risk). Configuration drift (If someone changes K8s manually, pipeline doesn't know).
*   **Pull-Based (GitOps):** An agent inside the cluster (ArgoCD) watches the Git repo. If Git changes, it updates the cluster.
    *   **Pros:** Secure (No external credentials). Self-healing (Reverts manual changes).

---

## 2. Advanced: RBAC for CI/CD (Push Model)
If you MUST use Push-Based, do not give `admin` access. Create a ServiceAccount.

### Step 1: Create ServiceAccount
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cicd-user
  namespace: default
```

### Step 2: Create Role (Permissions)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cicd-role
  namespace: default
rules:
- apiGroups: ["apps", ""]
  resources: ["deployments", "pods", "services"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

### Step 3: Bind Role to User
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cicd-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: cicd-user
  namespace: default
roleRef:
  kind: Role
  name: cicd-role
  apiGroup: rbac.authorization.k8s.io
```

---

## 3. Helm: The Package Manager for K8s
Writing raw YAMLs is tedious. Helm uses templates.
*   **Chart:** The package (folder structure).
*   **Values:** The configuration (`values.yaml`).
*   **Release:** A deployed instance of a chart.

### Chart Structure
```text
mychart/
  Chart.yaml          # Metadata (name, version)
  values.yaml         # Default values
  templates/          # Go Templates
    deployment.yaml
    service.yaml
```

### Templating Example (`deployment.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-app
spec:
  replicas: {{ .Values.replicaCount }}
```

### Commands
*   `helm install my-release ./mychart`
*   `helm upgrade my-release ./mychart --set replicaCount=5`
*   `helm rollback my-release 1`

---

## 4. Kustomize (Native Configuration Management)
Built into `kubectl`. Uses "Overlays" to patch base YAMLs.
*   **Base:** Common YAMLs.
*   **Overlays (Dev/Prod):** Patches (Change replicas, CPU limits).

**Structure:**
```text
base/
  deployment.yaml
  kustomization.yaml
overlays/
  dev/
    kustomization.yaml (patches base)
  prod/
    kustomization.yaml (patches base)
```

**Apply:** `kubectl apply -k overlays/prod`

---

## 5. GitOps with ArgoCD
1.  **Install ArgoCD:** `kubectl apply -f ...`
2.  **Define Application:**
    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: guestbook
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj/argocd-example-apps.git
        targetRevision: HEAD
        path: guestbook
      destination:
        server: https://kubernetes.default.svc
        namespace: guestbook
    ```
3.  **Sync:** ArgoCD pulls the repo and applies the manifests.
4.  **Drift Detection:** If you manually `kubectl delete service`, ArgoCD sees "OutOfSync" and recreates it (Self-Healing).

---

## 6. Interview Questions
1.  **Q: Difference between Helm and Kustomize?**
    *   *A:* **Helm** uses text templating (Go templates). Good for packaging and distribution. **Kustomize** uses structured patching (YAML merging). Good for managing variants (Dev/Prod) without complex logic.
2.  **Q: How do you handle secrets in GitOps?**
    *   *A:* You cannot store raw secrets in Git. Use **Sealed Secrets** (Bitnami) or **External Secrets Operator** (AWS Secrets Manager integration).
3.  **Q: What is a "Canary Analysis" in CD?**
    *   *A:* Using metrics (Prometheus) to decide if a Canary deployment is good. "If error rate < 1%, promote to 100% traffic. Else rollback." Tools: Flagger / Argo Rollouts.
