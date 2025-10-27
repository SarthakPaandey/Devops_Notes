# Day 1: Microservices Architecture & Design Principles
**Date:** 27th October

## 1. Typical Microservices Architecture
In a microservices architecture, the application is split into smaller, independent services. Unlike a Monolith, where everything runs in one process, microservices communicate over the network.

**Key Components:**
*   **Clients:** Mobile, Web, or 3rd party apps.
*   **Identity Provider:** Handles Auth (OAuth2/OIDC). Centralized user management (Keycloak/Auth0).
*   **API Gateway:** The single entry point. Handles routing, rate limiting, and SSL termination.
*   **Microservices:** The actual logic (Account Service, Inventory Service).
*   **Databases:** Private data stores for each service. **Database per Service** pattern is critical.

---

## 2. Infrastructure Components

### DNS (Domain Name System)
The phonebook of the internet.
*   **Role:** Translates `api.myapp.com` -> `203.0.113.5`.
*   **Record Types:**
    *   **A Record:** Maps name to IPv4.
    *   **AAAA Record:** Maps name to IPv6.
    *   **CNAME:** Maps name to another name (Alias).
    *   **MX Record:** Mail exchange.

### Load Balancers (LB)
Distributes incoming traffic across multiple healthy servers.
*   **L4 LB (Transport Layer):** Routes based on IP/Port (TCP/UDP). Very fast. (e.g., AWS NLB).
*   **L7 LB (Application Layer):** Routes based on URL path, Cookies, Headers (HTTP). Smarter but slower. (e.g., Nginx, AWS ALB).
*   **Health Checks:** The LB pings the server (`/health`). If 200 OK, it sends traffic. If 500/Timeout, it stops sending traffic.

---

## 3. Communication Patterns

### Synchronous (Blocking)
*   **Protocol:** HTTP/REST, gRPC.
*   **Flow:** Client sends request -> Waits -> Server processes -> Server responds.
*   **Pros:** Simple to understand. Real-time.
*   **Cons:** Coupling. If Service B is slow, Service A hangs.
*   **Use Case:** Real-time data retrieval (Get User Profile).

### Asynchronous (Non-Blocking)
*   **Protocol:** AMQP (RabbitMQ), Kafka.
*   **Flow:** Client sends message -> Server acks -> Worker processes later.
*   **Pros:** Decoupling. Service A can continue even if Service B is down.
*   **Cons:** Complexity (Eventual Consistency).
*   **Use Case:** Sending emails, generating reports, payment processing.

---

## 4. Data Storage: SQL vs NoSQL

| Feature | SQL (Relational) | NoSQL (Non-Relational) |
| :--- | :--- | :--- |
| **Structure** | Tables, Rows, Columns | Documents, Key-Value, Graphs |
| **Schema** | Strict/Fixed (Alter Table is hard) | Flexible/Dynamic (JSON) |
| **Scaling** | Vertical (Bigger Server/RAM) | Horizontal (Sharding/Partitioning) |
| **ACID** | Strong Compliance | Often Eventual Consistency (BASE) |
| **Examples** | MySQL, PostgreSQL | MongoDB, Cassandra, Redis |
| **Use Case** | Financial, Relations, Transanctions | Analytics, Catalog, Real-time |

---

## 5. CAP Theorem Deep Dive
In a Distributed System, you can only pick 2 out of 3:
1.  **Consistency (C):** Every read receives the most recent write or an error.
2.  **Availability (A):** Every request receives a (non-error) response, without the guarantee that it contains the most recent write.
3.  **Partition Tolerance (P):** The system continues to operate despite an arbitrary number of messages being dropped/delayed by the network.

**The Reality:** Network partitions (P) **will** happen. So you must choose between CP or AP.
*   **CP (Consistency + Partition Tolerance):** If the link breaks, refuse connections to ensure data is correct. (e.g., Banking).
*   **AP (Availability + Partition Tolerance):** If the link breaks, accept writes even if they might conflict later. (e.g., Social Media feeds).

---

## 6. Design Pattern: Circuit Breaker
Prevents cascading failures. If Service B is failing, Service A should stop calling it to give it time to recover.

**States:**
1.  **Closed:** Normal operation. Calls go through.
2.  **Open:** Threshold crossed (e.g., 50% errors). Calls fail immediately (Fast Fail).
3.  **Half-Open:** Timeout passed. Allow 1 test request. If success, go to Closed. If fail, go to Open.

**Code Example (Pseudo-Java/Resilience4j):**
```java
@CircuitBreaker(name = "backendA", fallbackMethod = "fallback")
public String doSomething() {
    return restTemplate.getForObject("http://backendA/api", String.class);
}

public String fallback(Exception e) {
    return "Backend is down, please try later.";
}
```

---

## 7. API Gateway
A server that acts as an API front-end, receiving API requests, enforcing throttling and security policies, passing requests to the back-end service, and then passing the response back to the requester.

**Key Functions:**
*   **Authentication/Authorization:** Validate JWT tokens.
*   **Rate Limiting:** Allow only 100 req/min/user.
*   **SSL Termination:** Decrypt HTTPS here to offload work from services.
*   **Response Caching:** Cache common responses.

**Example Config (Nginx as Gateway):**
```nginx
server {
    listen 80;
    location /api/v1/users {
        proxy_pass http://user-service:3000;
    }
    location /api/v1/orders {
        proxy_pass http://order-service:4000;
    }
}
```

---

## 8. Externalizing Logs
**"Treat logs as event streams."**
In a monolith, you log to a file `/var/log/app.log`. In microservices, containers die and lose files.
*   **Practice:** App writes to `STDOUT` / `STDERR`.
*   **Collection:** A log shipper (Fluentd/Promtail/Filebeat) runs as a DaemonSet. It grabs logs from the container runtime.
*   **Aggregation:** Sends them to a central store (Elasticsearch/Loki/Splunk).
*   **Visualization:** Developers search via Kibana/Grafana.

---

## 9. The 12-Factor Apps
A methodology for building modern, scalable SaaS apps.
1.  **Codebase:** One repo per app. Tracked in version control.
2.  **Dependencies:** Explicitly declare (package.json/pom.xml). Do not rely on system-wide packages.
3.  **Config:** Store in Environment Variables. Never commit secrets to code.
4.  **Backing Services:** Treat DBs/Queues as attached resources. Easily swappable.
5.  **Build, Release, Run:** Strictly separate stages. Build creates an immutable artifact (Docker Image).
6.  **Processes:** Execute the app as one or more stateless processes. State belongs in the DB/Redis.
7.  **Port Binding:** Export services via port binding (App listens on 8080). Don't rely on Tomcat injection.
8.  **Concurrency:** Scale out via the process model (Horizontal scaling).
9.  **Disposability:** Fast startup (fast boot) and graceful shutdown (SIGTERM handling).
10. **Dev/Prod Parity:** Keep development, staging, and production similar. Use Docker Compose locally.
11. **Logs:** Treat logs as event streams. Don't worry about log rotation in the app.
12. **Admin Processes:** Run admin/management tasks (db migration) as one-off processes.
