# Day 7: Docker Images & Storage Internals
**Date:** 12th November

## 1. Installing Docker
*   **Linux:** `curl -fsSL https://get.docker.com | sh`
*   **Mac/Windows:** Install "Docker Desktop".
*   **Verify:** `docker run hello-world`
    *   This pulls the `hello-world` image from Docker Hub if missing.
    *   Creates a container from it.
    *   Stream output to console.
    *   Exits.

## 2. Docker Images Deep Dive
An image is an **immutable, read-only template** used to generate containers.
It includes:
*   Application Code.
*   Libraries/Dependencies.
*   Runtime (JRE, Python).
*   Environment Variables.
*   Configuration Files.

### The Registry
*   **Docker Hub:** Default public registry.
*   **Private Registries:** AWS ECR, Azure ACR, Google GCR, Harbor (Self-hosted).
*   **Authentication:** `docker login <registry-url>`.

---

## 3. Internals of Docker Images

### The Layered Architecture
Docker images are built from a series of layers. Each instruction in a `Dockerfile` creates a layer.
*   **Base Image:** The starting point (e.g., `FROM ubuntu:20.04`).
*   **Intermediate Layers:** Changes (e.g., `RUN apt-get update`).
*   **Top Layer:** The Container Layer (Read-Write).

**Example:**
1.  `FROM ubuntu` (Layer A: 72MB)
2.  `RUN apt-get install python` (Layer B: 50MB)
3.  `COPY app.py .` (Layer C: 1KB)

### The Manifest & Digest
*   **Tag:** `mysql:5.7` -> Mutable pointer. Can be overwritten.
*   **Digest:** `mysql@sha256:45b...` -> Content Addressable Hash. Immutable. Secure.
    *   **Best Practice:** In production, pin by Digest to avoid "it worked yesterday" bugs.
*   **Manifest List (Multi-Arch):** A single tag `node:14` points to a Manifest List, which contains pointers to `linux/amd64`, `linux/arm64` (for Apple M1), etc. Docker CLI automatically pulls the correct architecture.

### Union File System (UnionFS)
Docker uses UnionFS to stack these layers into a single unified view.
*   **Storage Driver:** The technology that implements UnionFS.
    *   **Overlay2:** The current standard (Linux). Fast, efficient inode usage.
    *   **VFS:** Slow, naive (copies everything).
    *   **Devicemapper:** Deprecated.
*   **Efficiency:** If you have 10 images based on Ubuntu, the Ubuntu layer is stored **once** on disk (in `/var/lib/docker/overlay2`). All 10 images share the specific read-only layer.

### Copy-On-Write (CoW)
When a container starts, it adds a thin "Read-Write" layer on top.
*   **Read:** If you read `/etc/hosts`, Docker looks down through the layers until it finds it (usually in Base). Zero penalty.
*   **Write:** If you try to edit `/etc/hosts`:
    1.  Docker pauses the write.
    2.  Copies the file from the Read-Only lower layer to the top Read-Write layer.
    3.  Performs the write on the copy.
    4.  **Implication:** This is slow for heavy writes (Databases). **Always use Volumes for heavy I/O.**

---

## 4. Image Management Commands
*   `docker images`: List local cache.
*   `docker rmi <id>`: Delete image (Must stop containers first).
*   `docker image inspect <id>`: View JSON metadata (Entrypoint, Env Vars).
*   `docker history <image>`: Show the layers and size of each layer. Critical for debugging bloat.

### Cleaning Up
*   **Dangling Images (`<none>`):** Layers that have no relationship to a tagged image. Often caused by rebuilding an image (old layers become dangling).
*   **Unused Images:** Images that are tagged but not used by any container.
*   **Prune:**
    *   `docker image prune`: Remove dangling images.
    *   `docker system prune -a`: Remove ALL unused images (be careful!), stopped containers, and networks.

---

## 5. Security & Optimization
1.  **Distroless Images:**
    *   Images that contain *only* your app and its runtime dependencies. No shell, no package manager, no text editor.
    *   **Pros:** Smaller attack surface. Can't run `sh` exploits.
    *   **Example:** `gcr.io/distroless/java:11`
2.  **Alpine Linux:**
    *   Tiny (5MB) base image. Uses `musl` libc instead of `glibc`.
    *   **Cons:** Compatibility issues with some C-based Python/Node libraries.
3.  **Debian Slim:**
    *   Middle ground. Standard glibc but stripped of man-pages/docs. `python:3.9-slim`.

---

## 6. Offline Transfer (Air-Gapped)
How to move images across networks without internet?
1.  **Export:** `docker save -o image.tar my-image:v1` (Saves all layers/history).
2.  **Transfer:** USB / SCP.
3.  **Import:** `docker load -i image.tar`.

---

## 7. Interview Questions
1.  **Q: Difference between `docker save` and `docker export`?**
    *   *A:* `docker save` persists the **Image** (layers + history). `docker export` persists the **Container Filesystem** (flattened, no history, no CMD/ENTRYPOINT metadata).
2.  **Q: Where are Docker images stored on the host?**
    *   *A:* `/var/lib/docker/overlay2` (on most modern Linux distros).
3.  **Q: Why is my image size so large despite deleting files in a later layer?**
    *   *A:* UnionFS behavior. If Layer 1 adds a 1GB file, and Layer 2 deletes it, Layer 2 effectively just "hides" it. The 1GB blob is still in Layer 1 and part of the image size. You must delete the file in the *same* `RUN` instruction where you created it.
