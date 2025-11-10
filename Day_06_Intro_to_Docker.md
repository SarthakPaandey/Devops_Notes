# Day 6: Docker Architecture & The OCI Standard
**Date:** 10th November

## 1. Introduction to Containerization
Containerization is a lightweight form of virtualization. Ideally, an application should run the same behavior on a laptop as it does on a server.
*   **Virtual Machine (VM):**
    *   Hardware -> Hypervisor -> Guest OS -> Bin/Lib -> App.
    *   **Pros:** Strong isolation (Kernel separation).
    *   **Cons:** Heavy (GBs), slow boot (minutes).
*   **Container:**
    *   Hardware -> Host OS -> Container Runtime -> App.
    *   **Pros:** Instant boot (milliseconds), tiny footprint (MBs).
    *   **Cons:** Shared Kernel (Security risk if kernel exploitable).

## 2. What is Docker?
Docker is an open platform for developing, shipping, and running applications. It standardized the container format.
*   **Before Docker:** LXC (Linux Containers) existed but were hard to use.
*   **Docker's Innovation:** "Build once, run anywhere" via the Docker Image format.

## 3. Why do you need Docker?
*   **Consistency:** Eliminates "Works on my machine" syndrome.
*   **Speed:** CI/CD pipelines run faster with cached layers.
*   **Isolation:** Dependencies are packaged *inside* the container (`/app/node_modules`), not on the host. You can run Node 12 and Node 18 apps side-by-side.
*   **Efficiency:** Higher density. A single 8GB RAM server can run 50 containers.

---

## 4. Docker Architecture
Docker uses a client-server architecture.

### Components
1.  **Docker Client:** The CLI tool (`docker build`, `docker run`). It talks to the Daemon via REST API (Unix Socket `/var/run/docker.sock`).
2.  **Docker Host:** The machine running the Daemon.
    *   **Docker Daemon (`dockerd`):** The heavy lifter. Manages images, containers, networks, volumes.
    *   **Containerd:** The high-level runtime. Manages the lifecycle (start/stop/pull). Used by Kubernetes (CRI).
    *   **Runc:** The low-level runtime. Actually interacts with the Kernel to create the process.
    *   **Shim:** Decouples the running container from the Daemon. Allows "Daemonless" upgrades (you can restart Docker Daemon without killing containers).
3.  **Docker Registry:** Stores images (Docker Hub, ECR, GCR).

### The OCI (Open Container Initiative)
Docker donated its specs to the OCI to prevent vendor lock-in.
1.  **Image Spec:** How layers and JSON config are packaged.
2.  **Runtime Spec:** How arguments, env vars, and mounts are passed to the container.

---

## 5. Security Internals (Kernel Primitives)

### Namespaces (Isolation - "What you SEE")
*   **PID:** Process IDs. Container PID 1 is Host PID 12345.
*   **NET:** Network stack (IP, Ports, Routing Table).
*   **MNT:** Filesystem mounts (`/`).
*   **UTS:** Hostname.
*   **IPC:** Inter-Process Communication (Shared Memory).
*   **USER:** UID/GID mapping (Map Host Root to Container Nobody).

### CGroups (Limits - "What you USE")
*   **Memory:** Limit RAM usage. OOM Kill if exceeded.
*   **CPU:** Limit cycles/shares.
*   **Block I/O:** Limit disk read/write speed.

### Capabilities (Privileges)
*   **Root inside container != Root on Host.**
*   Docker drops dangerous capabilities by default (e.g., `CAP_SYS_BOOT` - you can't reboot the host from a container).
*   **Privileged Mode (`--privileged`):** Gives all capabilities. **Avoid unless necessary.**

### Seccomp (Secure Computing Mode)
*   Whitelists/Blacklists System Calls.
*   Example: Block `chmod` or `chown` syscalls if the app doesn't need them.

### AppArmor / SELinux
*   Mandatory Access Control profiles. Restricts file access paths.

---

## 6. Alternative Runtimes
*   **gVisor (Google):** Security-focused. Intercepts syscalls in userspace. Stronger isolation, slight performance hit.
*   **Kata Containers:** Lightweight VMs as containers. Even stronger isolation (separate kernel).
*   **Podman (RedHat):** Daemonless. Rootless by default. OCI compliant drop-in replacement for Docker.

---

## 7. Interview Questions
1.  **Q: Difference between Docker and Virtual Machine?**
    *   *A:* VM has its own Guest OS Kernel (Heavy). Docker shares the Host OS Kernel (Light). VM = House; Container = Apartment in a building.
2.  **Q: What is the role of `containerd`?**
    *   *A:* It sits between the Daemon and Runc. It handles image pull/push and container supervision. Kubernetes bypasses Docker Daemon and talks directly to containerd (via CRI).
3.  **Q: Can you run Windows containers on Linux?**
    *   *A:* No (natively). Containers share the kernel. Linux containers need Linux Kernel. To run Windows containers on Linux, you need a VM wrapper (VirtualBox/KVM).
