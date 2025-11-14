# Day 8: Mastering Dockerfiles & Build Optimization
**Date:** 14th November

## 1. Executive Summary
A `Dockerfile` is a script of instructions. Just like code, it can be unoptimized, insecure, and buggy. Mastering the Dockerfile means mastering **Layer Caching** and **Image Size**.

---

## 2. Deep Dive: Instructions & Best Practices

| Instruction | Explanation | Best Practice |
| :--- | :--- | :--- |
| **FROM** | Base Image. | Use specific tags (`python:3.9-slim`). Avoid `latest`. |
| **COPY** | Copy code. | Use over `ADD` (unless fetching URL). `COPY . .` |
| **RUN** | Run build commands. | Chain with `&&` to reduce layers. |
| **WORKDIR** | Change directory. | Use absolute paths (`/app`). Don't use `cd`. |
| **CMD** | Default arg. | Can be overridden. `CMD ["python", "app.py"]` |
| **ENTRYPOINT** | Main executable. | Hard to override. `ENTRYPOINT ["python"]` |
| **USER** | Set user ID. | Run as non-root (security). |
| **ENV** | Set env vars. | Persist across layers. |
| **ARG** | Build-time variables. | Only available during build. |

---

## 3. Build Context Optimization
```bash
docker build -t app:v1 .
```
The `.` is the context. Docker zips *everything* in this folder and sends it to the Daemon.
*   **Problem:** If you have a 500MB `.git` folder or `node_modules`, the build is slow.
*   **Solution:** `.dockerignore` file.
    ```text
    .git
    node_modules
    Dockerfile
    *.md
    ```

---

## 4. Core Concept: Layer Caching
Docker caches the result of each line. If a line hasn't changed, it uses the cached layer.
**Order Matters:**
```dockerfile
# BAD: Breaks cache for npm install every time application code changes
COPY . .
RUN npm install

# GOOD: Only runs npm install if package.json changes
COPY package.json .
RUN npm install
# Now copy source code
COPY . .
```

---

## 5. Advanced: Multi-Stage Builds
Build your artifact in a heavy image (SDK), copy just the binary to a tiny image (Runtime).

### Example: Golang Application
```dockerfile
# --- Stage 1: Build (Golang Image - 800MB) ---
FROM golang:1.16 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o myapp .

# --- Stage 2: Run (Alpine - 5MB) ---
FROM alpine:latest
WORKDIR /root/
# Copy ONLY the binary file
COPY --from=builder /app/myapp .
# Install certificates (minimal dependency)
RUN apk --no-cache add ca-certificates
CMD ["./myapp"]
```

### Example: Java Spring Boot
```dockerfile
# --- Stage 1: Build (Maven - 600MB) ---
FROM maven:3.8-jdk-11 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

# --- Stage 2: Run (JRE - 200MB) ---
FROM openjdk:11-jre-slim
WORKDIR /app
COPY --from=build /app/target/app.jar app.jar
# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## 6. Linting Dockerfiles
Just like ESLint for JS, use **Hadolint** for Dockerfiles to catch bad practices.
*   `hadolint Dockerfile`
    *   *Warning:* DL3008: Pin versions in `apt-get install <package>=<version>`.
    *   *Warning:* DL3009: Delete the apt-get lists after installing (to save space).

---

## 7. Security Best Practices
1.  **Don't Run as Root:**
    By default, containers run as root. If the container is hacked, the attacker has root on the container.
    ```dockerfile
    RUN useradd -m appuser
    USER appuser
    ```
2.  **Use Trusted Base Images:**
    Only pull from Official Docker Hub repos or your company's private registry.
3.  **Scan Images:**
    Integrate scanning tools like **Trivy** or **Snyk** in your CI pipeline.

---

## 8. Interview Questions
1.  **Q: What is the difference between `ADD` and `COPY`?**
    *   *A:* `COPY` only copies local files. `ADD` can download URLs and auto-extract tarballs (Magic behavior). Use `COPY` unless you need the magic.
2.  **Q: Why do we chain `RUN` commands with `&&`?**
    *   *A:* To create a single layer. `RUN apt-get update && apt-get install curl` prevents an issue where a cached `update` layer causes `install` to fail with outdated indices.
3.  **Q: What is the PID 1 problem in Docker?**
    *   *A:* If your app runs as PID 1, it might not handle System Signals (SIGTERM) correctly, meaning `docker stop` waits 10s and then force kills it. Use `tini` (`--init` flag) or ensure your app handles signals properly.
4.  **Q: Explain how `ONBUILD` works?**
    *   *A:* An instruction that executes *when the image is used as a base for another build*. Useful for creating boilerplates (e.g., a standard Python Flask base image that auto-installs requirements if present).
