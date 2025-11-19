# Day 10: Data Persistence, Backup Strategies & Volume Internals
**Date:** 19th November

## 1. Introduction to Docker Volumes
Containers are ephemeral (Temporary). **Volumes** are the Docker mechanism to ensure data survives even if the container is destroyed. Without volumes, your database is just an empty shell waiting to lose data.

## 2. Types of Volumes Deep Dive

### A. TmpFs (Temporary File System)
*   **Stored in:** RAM (Memory).
*   **Behavior:** Never writes to disk.
*   **Use Case:** Storing secrets, tokens, or encryption keys that should never touch the hard drive (Compliance).
*   **Flags:** `--tmpfs /app/secrets:rw,noexec,nosuid,size=64m`.
    *   `noexec`: Prevent execution of binaries (Security).
    *   `size`: Limit RAM usage.

### B. Bind Mounts (The "Dev Standard")
*   **Location:** Any path you specify (e.g., `/home/user/code`).
*   **Behavior:** Maps a Host directory to a Container directory.
*   **Pros:** Changes on Host are instantly visible in Container.
*   **Cons:** Tightly coupled to Host OS paths. Permissions hell (UID mismatch).
*   **Propagation:** Should a mount inside the host folder be visible inside the container?
    *   `rprivate` (Default): No.
    *   `rshared`: Yes. (Advanced).

### C. Docker Volumes (The "Prod Standard")
*   **Location:** `/var/lib/docker/volumes/my-vol/_data` (Managed by Daemon).
*   **Pros:** Easy to backup, works on Windows/Mac/Linux identically, supports Volume Drivers (Cloud Storage).
*   **Commands:**
    ```bash
    docker volume create db-data
    docker volume inspect db-data
    docker volume prune  # Delete unattached volumes
    ```

---

## 3. Deep Dive: Storage Mechanics & Drivers

### UnionFS vs Volumes
*   **Container Root Filesystem:** Uses Overlay2 (UnionFS).
    *   **Performance Penalty:** Copy-on-Write (CoW). To write a 1GB file, Docker first copies it from the lower layer. Slow for Databases.
*   **Volumes:** Bypass the UnionFS and access disk directly.
    *   **Performance:** Near-native speed. Zero CoW overhead.
    *   **Conclusion:** ALWAYS use volumes for I/O heavy workloads (Postgres, MySQL, Kafka).

### Volume Drivers
You typically use the `local` driver. But you can install plugins:
*   **RexRay / Portworx:** Create volumes on AWS EBS / Azure Disk.
    `docker volume create -d rexray/ebs --name my-ebs-vol -o size=10`

---

## 4. Production Scenarios: Operations

### Backup & Restore
How do you backup a volume? You can't just copy the folder while the DB is running (Corruption risk).
**The "Tarball Sidecar" Pattern:**
1.  Stop the container (or lock tables).
2.  Run a temporary "Helper" container that mounts the volume AND a backup directory.
3.  Tar the volume content.
```bash
# Backup Command
docker run --rm \
  -v db-data:/volume \
  -v $(pwd):/backup \
  ubuntu \
  tar -cvf /backup/backup.tar /volume
```
_This creates `backup.tar` in your current directory._

**Restore Command:**
```bash
docker run --rm \
  -v db-data:/volume \
  -v $(pwd):/backup \
  ubuntu \
  bash -c "cd /volume && tar -xvf /backup/backup.tar --strip 1"
```

### Permission Issues (UID Mismatch)
*   **Scenario:** You bind mount a folder, but the app crashes with `Permission Denied`.
*   **Cause:** Host user is UID 1000. Container user is UID 1000. If they match, it works. If container runs as UID 999 (postgres) and folder is owned by root, it fails.
*   **Fix:** `chown` the host directory `chown 999:999 data/` or force container UID with `--user`.

---

## 5. Cleaning Up (Garbage Collection)
Docker volumes are **never** deleted automatically when a container is removed.
*   **Good:** Data safety.
*   **Bad:** Disk fills up with orphaned volumes.
*   **Solution:**
    *   `docker rm -v <container_id>`: Remove volume associated with container (if anonymous).
    *   `docker volume prune`.

---

## 6. Interview Questions
1.  **Q: Can I share a volume between multiple containers?**
    *   *A:* Yes. Example: A "Log Producer" container writes to `/logs` (Volume A). A "Splunk Agent" container reads from `/logs` (Volume A) in Read-Only mode (`:ro`).
2.  **Q: What happens if I mount a volume to a container directory that already has files?**
    *   *A:* The files in the container image are **hidden** (masked) by the volume mount. They are not deleted, just inaccessible until you unmount.
3.  **Q: Why use a Volume instead of a Bind Mount in Production?**
    *   *A:* Portability. Bind mounts rely on specific host paths (`/home/deploy/app`), which might not exist on all servers. Named Volumes abstract the location away. Also, Volumes mimic native performance better.
4.  **Q: How do you migrate a volume from Host A to Host B?**
    *   *A:* `docker run ... tar` (Backup) -> SCP tarball -> `docker run ... untar` (Restore). Or use a Cloud Volume Driver (EBS/NFS) so both hosts can see the same disk.
