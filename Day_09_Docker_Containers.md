# Day 9: Docker Container Deep Dive & Resource Management
**Date:** 17th November

## 1. What is a Container Internally?
A container is not a physical object; it is a **Process** with isolation constraints applied key kernel features.
*   **Command:** `docker run ubuntu sleep 100` helps visualize this.
*   **Host view:** `ps aux | grep sleep` shows a process running on the host kernel.
*   **Container view:** `docker exec ... ps aux` shows PID 1.

## 2. Kernel Primitives (The Magic)

### Namespaces (Isolation - "What you SEE")
Namespaces partition kernel resources so one set of processes sees one set of resources while another set of processes sees a different set.
1.  **PID:** Process Isolation. The container thinks it is PID 1.
2.  **NET:** Network Interfaces (eth0), Ports.
3.  **MNT:** File System Mounts.
4.  **UTS:** Hostname.
5.  **IPC:** Inter-Process Communication (Shared Memory).
6.  **USER:** UID/GID Mapping.

### CGroups (Control Groups - "What you USE")
Control Groups limit, account for, and isolate the resource usage (CPU, memory, disk I/O, network) of a collection of processes.

#### Memory Constraints
By default, a container has **unlimited** access to the Host's RAM. One memory leak can crash the whole server.
*   `--memory="512m"`: Hard limit (RAM).
    *   **OOM Killer (Out Of Memory):** If a container exceeds the limit, the Kernel sends `SIGKILL` (Signal 9) to the process.
    *   **Debug:** Check `dmesg | grep -i "killed process"` on the host to see OOM kills.
*   `--memory-swap`: Controls swap usage.

#### CPU Constraints
*   `--cpus="1.5"`: Guarantees the container can use at most 1.5 cores.
*   `--cpu-shares=512`: Soft limit. Prioritizes CPU cycles relative to other containers (Default is 1024).

---

## 3. Container Lifecycle States
1.  **Created:** Container created but not started (`docker create`).
2.  **Running:** Process is active (`docker start`).
3.  **Paused:** Processes frozen. Memory state preserved (`docker pause`).
4.  **Exited:** Process finished. Exit Code 0 (Success) or Non-Zero (Error).
5.  **Dead:** Daemon tried to kill it but failed (Zombie state).
6.  **Removing:** Being deleted.

### The "Crash Loop BackOff"
Your container starts and immediately dies. `docker logs` shows nothing.
**Steps:**
1.  **Override Entrypoint:**
    ```bash
    docker run -it --entrypoint /bin/sh my-app
    ```
    This lets you explore the filesystem before the app crashes.
2.  **Inspect State:**
    ```bash
    docker inspect my-app | grep ExitCode
    ```
    *   `137`: OOM Killed (Ran out of RAM).
    *   `127`: Command not found.
    *   `1`: App error.

---

## 4. Copy-On-Write (CoW) & Performance
How does CoW help?
*   **Storage Efficiency:** 100 containers share the same Read-Only Ubuntu image (200MB). They don't take 20GB space; they take 200MB + tiny R/W layers.
*   **Startup Speed:** No need to copy files. Just create a pointer to the existing image layers. This is why containers start in milliseconds.

---

## 5. Security Context
By default, Docker containers run as root.
*   **User namespace remapping:** Maps container root (UID 0) to host non-privileged user (UID 1000).
*   **Seccomp:** Whitelist syscalls.
*   **Capabilities:** `docker run --cap-drop ALL --cap-add NET_BIND_SERVICE ...`

---

## 6. Interview Questions
1.  **Q: What happens if I don't set memory limits?**
    *   *A:* The container can consume all available host RAM. If it leaks memory, it will trigger the Host OOM Killer, which might kill crucial system processes (like `sshd` or `dockerd`), bringing down the node.
2.  **Q: Explain the `docker0` bridge.**
    *   *A:* It's a virtual ethernet bridge created by Docker on startup. It sits between the physical interface (eth0) and the container interfaces (veth pairs), performing NAT routing.
3.  **Q: Difference between `docker pause` and `docker stop`?**
    *   *A:* `pause` freezes the process using cfreezer cgroup (RAM stays in use). `stop` sends SIGTERM, waits 10s, then SIGKILL (RAM interacts with disk/swap).
