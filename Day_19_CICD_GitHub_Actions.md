# Day 19: CI/CD with GitHub Actions Advanced
**Date:** 17th December

## 1. What is CI/CD?
*   **Continuous Integration (CI):** Developers commit code -> Automated Build & Test.
*   **Continuous Deployment (CD):** Validated code -> Auto-deploy to Production.

## 2. GitHub Actions Concepts
A CI/CD platform built into GitHub.
*   **Workflow:** A configurable automated process defined by a YAML file (`.github/workflows/main.yml`).
*   **Jobs:** A workflow run is made up of one or more jobs. Jobs run in parallel by default.
*   **Steps:** A sequence of tasks (commands/actions) inside a job. Sequential.
*   **Context:** `github.event`, `github.sha`, `github.ref`. Access metadata.

---

## 3. Workflow Syntax Deep Dive
```yaml
name: CI Pipeline
on: 
  push:
    branches: [ "main" ]
  pull_request:
    types: [opened, synchronize, reopened]

env:
  NODE_ENV: production

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix: # Run across multiple versions
        node-version: [14.x, 16.x]
    steps:
    - uses: actions/checkout@v3 # Clone repo
    
    - name: Use Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v3
      with:
        node-version: ${{ matrix.node-version }}
        cache: 'npm' # Auto-caching!
        
    - run: npm ci
    - run: npm run build --if-present
    - run: npm test
```

### Contexts & Secrets
*   `${{ github.sha }}`: Commit Hash.
*   `${{ secrets.AWS_ACCESS_KEY_ID }}`: Secure variable (Settings -> Secrets).
*   `${{ env.NODE_ENV }}`: Environment variable.

---

## 4. Reusable Workflows (`workflow_call`)
Don't copy-paste YAML. Create a template.
**Template (`.github/workflows/reusable-build.yml`):**
```yaml
name: Reusable Build
on:
  workflow_call:
    inputs:
      image-name:
        required: true
        type: string
    secrets:
      token:
        required: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building ${{ inputs.image-name }}"
```

**Caller:**
```yaml
jobs:
  call-build:
    uses: ./.github/workflows/reusable-build.yml
    with:
      image-name: my-app
    secrets:
      token: ${{ secrets.GITHUB_TOKEN }}
```

---

## 5. Composite Actions
Create your own action without Docker/JS. Just a `action.yml`.
**File: `.github/actions/setup-python/action.yml`**
```yaml
name: 'Setup Python & Poetry'
description: 'Custom setup'
runs:
  using: "composite"
  steps:
    - uses: actions/setup-python@v4
      with:
        python-version: '3.10'
    - run: pip install poetry
      shell: bash
```

---

## 6. Advanced Caching
Dependencies (node_modules) take time to download. Cache them.
```yaml
- name: Cache node modules
  uses: actions/cache@v3
  env:
    cache-name: cache-node-modules
  with:
    path: ~/.npm
    key: ${{ runner.os }}-build-${{ env.cache-name }}-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-build-${{ env.cache-name }}-
```

---

## 7. Improving CI: DevSecOps (SCA & SAST)
Add security scans to the pipeline.

### SCA (Software Composition Analysis)
Checks for dependencies with known vulnerabilities.
**Tool:** Trivy (File System mode).

### SAST (Static Application Security Testing)
Checks source code for bad patterns.
**Tool:** SonarQube or CodeQL.

**Updated YAML:**
```yaml
jobs:
  security:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    # 1. SCA Scan (Trivy)
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        ignore-unfixed: true
        severity: 'CRITICAL,HIGH'

    # 2. Build Docker Image
    - name: Build Docker Image
      run: docker build -t myapp:${{ github.sha }} .
```

---

## 8. Interview Questions
1.  **Q: Difference between `inputs` and `secrets` in reusable workflows?**
    *   *A:* **Inputs** are plain text config logic. **Secrets** are masked in logs and encrypted.
2.  **Q: How do you trigger a workflow manually?**
    *   *A:* `on: workflow_dispatch`. Adds a "Run workflow" button in UI.
3.  **Q: What happens if I push to a branch named `gh-pages`?**
    *   *A:* By default, GitHub looks for a Jekyl build. You can override this behavior with a custom workflow file `pages.yml`.
4.  **Q: How do you share data between jobs?**
    *   *A:* **Artifacts.** `actions/upload-artifact` in Job A. `actions/download-artifact` in Job B.
