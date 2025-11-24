# Day 12: Kubernetes Architecture Deep Dive
**Date:** 24th November

## 1. The Need for Orchestration
Managing 10 containers is easy (`docker-compose`). Managing 10,000 containers across 50 servers is hard.
**Problems Solved by Orchestration:**
*   **Scheduling:** "Put this container on Server 50 because it has 2GB RAM free".
*   **Self-Healing:** "Server 50 died? Move all its containers to Server 51".
*   **Scaling:** "Traffic is high? Start 5 more Nginx containers".
*   **Load Balancing:** "Distribute traffic evenly to all 5 containers".
*   **Updates:** "Update image v1 to v2 without downtime".

## 2. Kubernetes (K8s) Overview
Greek for "Helmsman" (Pilot of the ship). Created by Google (borg), donated to CNCF.
*   **Declarative Model:** You tell K8s "I want 3 Nginx pods" (Desired State). K8s ensures "There are 3 Nginx pods" (Current State).

---

## 3. Architecture: The Control Plane (Master Node)
The Brain. Makes global decisions. In HA (High Availability), you run 3 or 5 masters.

### 1. API Server (`kube-apiserver`)
*   **Role:** The Frontend / Gatekeeper.
*   **Function:** Validates requests, authenticates users, and updates the database.
*   **Feature:** The ONLY component that talks to Etcd.
*   **Scaling:** Stateless. Can run multiple instances behind a Load Balancer.

### 2. Etcd
*   **Role:** The Database (Source of Truth).
*   **Function:** Consistent, highly-available Key-Value store. Stores cluster state (Pods, Secrets, Configs).
*   **Quorum:** Needs (N/2)+1 nodes to write. If you lose quorum, the cluster goes Read-Only.

### 3. Scheduler (`kube-scheduler`)
*   **Role:** The Matchmaker.
*   **Function:** Watches for new Pods with no assigned Node. Selects a Node based on:
    *   Resource Requirements (CPU/RAM).
    *   Taints/Tolerations.
    *   Affinity/Anti-Affinity rules.

### 4. Controller Manager (`kube-controller-manager`)
*   **Role:** The Fixer.
*   **Function:** Runs control loops.
    *   **Node Controller:** Notices if a node goes down.
    *   **Replication Controller:** Maintains correct number of pods.
    *   **Endpoint Controller:** Populates Service objects.

### 5. Cloud Controller Manager (CCM)
*   **Role:** The Cloud Interface.
*   **Function:** Talks to AWS/Azure/GCP APIs to creates Load Balancers and EBS Volumes.

---

## 4. Architecture: The Worker Node
The Muscle. Runs the actual applications.

### 1. Kubelet
*   **Role:** The Agent (Captain).
*   **Function:**
    *   Registers Node with API Server.
    *   Listens for instructions ("Run Pod A").
    *   Instructs Runtime to start container.
    *   Reports health (`LivenessProbe`) back to Master.

### 2. Kube-Proxy
*   **Role:** The Networker.
*   **Function:** Maintains network rules (IPTables / IPVS) on the node. Allows Pods to talk to Services.

### 3. Container Runtime
*   **Role:** The Engine.
*   **Function:** Pulls images and runs containers.
*   **CRI (Container Runtime Interface):** K8s interface.
    *   Docker (via Dockershim - Deprecated).
    *   Containerd (Industry Standard).
    *   CRI-O (RedHat).

---

## 5. Kubernetes Distributions
Not all K8s are the same.
1.  **Kubeadm:** The "Hard Way" (but automated). Vanilla upstream K8s. Good for learning/Certifications (CKA).
2.  **Minikube / Kind:** Single-node cluster for laptops. Runs inside Docker.
3.  **K3s:** Lightweight (IoT/Edge). Single binary (<50MB).
4.  **Managed (EKS/AKS/GKE):** Cloud provider manages Control Plane. You manage Workers.

---

## 6. The "Pause" Container
Every Pod has a hidden "Pause" container (sandbox).
*   **Purpose:** Holds the Network Namespace and Volume Mounts.
*   **Why:** If the main application container crashes and restarts, the Pause container stays alive, keeping the IP address same.

---

## 7. Interview Questions
1.  **Q: If the Master Node goes down, do my applications stop?**
    *   *A:* **NO.** The applications run on Worker Nodes. They continue to serve traffic. However, you cannot *change* anything (deploy new apps, scale up) until Master is back. Self-healing (restarting crashed pods) also stops.
2.  **Q: What is the difference between Imperative and Declarative?**
    *   *A:* **Imperative:** "Run this command" (`kubectl run nginx`). **Declarative:** "Here is a file describing the final state" (`kubectl apply -f nginx.yaml`). K8s prefers Declarative.
3.  **Q: Why do we need `etcd`? Why not SQL?**
    *   *A:* K8s needs a distributed consistency system that handles network partitions well (CAP theorem). Etcd uses the Raft protocol to ensure strict consistency (CP system) which is critical for cluster state.
4.  **Q: What happens if `kubelet` dies?**
    *   *A:* The Node becomes `NotReady`. The Control Plane waits for `pod-eviction-timeout` (usually 5 mins). If Kubelet doesn't return, Scheduler reschedules all pods to other healthy nodes.
