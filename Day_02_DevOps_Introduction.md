# Day 2: Introduction to DevOps & DevSecOps
**Date:** 29th October

## 1. What is DevOps?
DevOps is a set of practices, cultural philosophies, and tools that integrate and automate the work of software development (Dev) and IT operations (Ops) to improve and shorten the systems development life cycle.
It's about removing the silos between "Idea" and "Production".

## 2. Why do you need DevOps?
*   **Speed:** Move at high velocity so you can innovate for customers faster.
*   **Reliability:** Ensure updates are safe (CI/CD) and infrastructure is stable.
*   **Scale:** Manage complex systems efficiently.
*   **Collaboration:** Build more effective teams under a cultural model.
*   **Security:** Integrate security checks into the pipeline (DevSecOps).

---

## 3. DevOps Lifecycle (The Infinity Loop)
The lifecycle is a continuous loop:
1.  **Plan:** Define features and requirements (Jira).
2.  **Code:** Write code and configuration (Git).
3.  **Build:** Compile and package (Maven/Docker). Create an artifact.
4.  **Test:** Automated testing (Selenium, JUnit, Unit Integration).
5.  **Release:** Manage release versions (SemVer).
6.  **Deploy:** To production environments (Kubernetes/Ansible).
7.  **Operate:** Manage infrastructure (Terraform).
8.  **Monitor:** Check performance and logs (Prometheus/Grafana).

---

## 4. DevOps Principles (CAMS)
*   **Culture:** Foster collaboration and shared responsibility. No blame game. Focus on "How do we prevent this next time?"
*   **Automation:** Eliminate manual toil. Humans make mistakes; scripts don't.
*   **Measurement:** Data-driven decisions. Determine success by metrics, not gut feeling.
*   **Sharing:** Open information flow (Wikis, ChatOps).

---

## 5. DORA Metrics (Measuring DevOps Success)
Google's DevOps Research and Assessment (DORA) identified 4 key metrics:
1.  **Deployment Frequency (Velocity):** How often do you ship to production?
    *   *Elite:* On-demand (multiple deploys per day).
    *   *Low:* Once every 6 months.
2.  **Lead Time for Changes (Velocity):** Time from "Commit" to "Running in Production".
    *   *Elite:* Less than 1 hour.
3.  **Change Failure Rate (Stability):** Percentage of deployments causing a failure in production.
    *   *Elite:* 0-15%.
4.  **Time to Restore Service (Stability):** How long to recover from a failure?
    *   *Elite:* Less than 1 hour.

---

## 6. DevOps Practices Deep Dive
*   **Continuous Integration (CI):** Merging code into main branch often (daily). Automated builds and tests verify the merge.
*   **Continuous Delivery (CD):** Always ready to deploy. The release to production is a manual button click.
*   **Continuous Deployment:** The release to production is automated if tests pass.
*   **Infrastructure as Code (IaC):** Managing infrastructure (servers, networks) using configuration files rather than GUI/Manual clicks.
*   **GitOps:** Using Git as the single source of truth for declarative infrastructure and applications. (e.g., ArgoCD).
*   **Monitoring & Logging:** Real-time visibility into application performance.

---

## 7. DevSecOps: Security in the Pipeline
Shift Left Security: Testing early in the process.

### SAST (Static Application Security Testing)
*   Scanning source code for vulnerabilities.
*   **When:** During Commit/PR.
*   **Tools:** SonarQube, CodeQL.

### DAST (Dynamic Application Security Testing)
*   Attacking the running application from outside.
*   **When:** In Staging/QA Environment.
*   **Tools:** OWASP ZAP, Burp Suite.

### SCA (Software Composition Analysis)
*   Scanning dependencies (libraries) for known CVEs.
*   **When:** During Build.
*   **Tools:** Snyk, Trivy, Dependabot.

---

## 8. Tools in DevOps Ecosystem
| Category | Tool Examples | Purpose |
| :--- | :--- | :--- |
| **Version Control** | Git, GitHub, GitLab | Managing code history & collaboration. |
| **CI/CD** | Jenkins, GitHub Actions, CircleCI | Automating build, test, and deploy pipelines. |
| **Containers** | Docker, Podman | Packaging applications with dependencies. |
| **Orchestration** | Kubernetes, Docker Swarm | Managing container scaling and networking. |
| **IaC** | Terraform, Ansible, Pulumi | Provisioning cloud resources programmatically. |
| **Monitoring** | Prometheus, Grafana, ELK Stack | Metrics collection and visualization. |
| **Artifact Repo** | Nexus, Artifactory, ECR | Storing built binaries (JARs, Docker Images). |
| **Communication** | Slack, Teams, Jira | Collaboration and issue tracking. |

## 9. Interview Questions
1.  **Q: Explain the difference between Continuous Delivery and Deployment.**
    *   *A:* **Delivery:** Code is deployable, but requires manual approval (Button click). **Deployment:** Fully automated to Prod.
2.  **Q: How do you handle a failed deployment?**
    *   *A:* Use **Rollbacks**. If using Blue/Green, switch traffic back. If Rolling, revert image tag. Ideally, automated health checks should abort the deployment before it finishes.
3.  **Q: What is "Shift Left"?**
    *   *A:* Moving testing and security checks earlier in the pipeline (to the left on the timeline), finding bugs when they are cheapest to fix.
