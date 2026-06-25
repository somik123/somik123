# GitHub Actions Cheat Sheet  

---

## Quick Reference

| Feature | Syntax (YAML) | Example |
|---------|---------------|---------|
| **Workflow trigger** | `on:` | `on: push` |
| **Run on specific branches** | `branches:` | `branches: [main, develop]` |
| **Schedule (cron)** | `schedule:` | `- cron: '0 2 * * *'` |
| **Job name & runner** | `jobs.<job_id>.runs-on:` | `runs-on: ubuntu-latest` |
| **Steps** | `steps:` | `- name: Checkout code\n  uses: actions/checkout@v4` |
| **Run a script** | `run:` | `run: echo "Hello, world!"` |
| **Using an action** | `uses:` | `uses: actions/setup-node@v4` |
| **Set env vars** | `env:` | `env:\n  NODE_ENV: production` |
| **Secrets** | `${{ secrets.NAME }}` | `${{ secrets.GITHUB_TOKEN }}` |
| **Artifacts** | `actions/upload-artifact@v3` | `- uses: actions/upload-artifact@v3\n  with:\n    name: my-artifact\n    path: ./dist/` |
| **Matrix builds** | `strategy.matrix:` | `matrix:\n  node-version: [14, 16, 18]` |

---

## Trigger (`on:`)

| Trigger | Description | Possible Values |
|---------|-------------|-----------------|
| `push` | When commits are pushed to a branch. | `<branch pattern>`, `<tag pattern>`, `paths`, `branches-ignore`, `tags-ignore`, `pull_request` (see below). |
| `pull_request` | On PR creation, update, or merge. | Same as `push`. |
| `workflow_dispatch` | Manual run via UI. | `inputs:` – key/value pairs; each input can have a `description`, `required`, `default`. |
| `repository_dispatch` | Custom event from API. | `types:` – list of strings that identify the event. |
| `schedule` | Cron schedule (UTC). | `cron: "0 5 * * *"` |
| `release` | On release actions. | `types:` – `published`, `unpublished`, `prereleased`, etc. |
| `check_suite` / `check_run` | When a check suite or run completes. | `types:` – `completed`, `requested_action`. |
| `issue_comment`, `issues`, `label`, `project`, `workflow_call` … | Many more built‑in events; each has its own sub‑fields. |

> **Branch / Tag patterns**  
> - Use GitHub’s *refspec* syntax: `main`, `develop`, `release/*`, `v[0-9]+.[0-9]+.*`.  
> - `branches:` and `tags:` accept a list or a single string.  
> - Wildcards (`*`) are allowed; double‑asterisk (`**`) matches multiple path segments.

---

## Basic Workflow Example

```yaml
name: CI

on:
  push:
    branches: [ main ]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repo
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Upload coverage report
        if: success()
        uses: actions/upload-artifact@v3
        with:
          name: coverage
          path: ./coverage/
```

---

## Advanced Features

### A. **Matrix Builds**

Run the same job across multiple OSes, languages, or config options.

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node-version: [14, 16, 18]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm ci && npm test
```

### B. **Caching Dependencies**

```yaml
- name: Cache Node modules
  uses: actions/cache@v3
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-
```

### C. **Conditional Execution**

```yaml
steps:
  - name: Deploy to prod
    if: github.ref == 'refs/heads/main' && success()
    run: ./deploy.sh
```

### D. **Reusable Workflows (Composite Actions)**

Create a reusable workflow in `.github/workflows/reusable.yml`:

```yaml
name: Reusable Test Suite
on:
  workflow_call:
    inputs:
      node-version:
        required: true
        type: string

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
      - run: npm ci && npm test
```

Invoke it from another workflow:

```yaml
jobs:
  call-test:
    uses: ./.github/workflows/reusable.yml
    with:
      node-version: '18'
```

### E. **Secrets & Contexts**

| Context | Example |
|---------|---------|
| `secrets` | `${{ secrets.S3_KEY }}` |
| `env` (repo level) | `env:\n  APP_ENV: prod` |
| `github` | `${{ github.repository }}`, `${{ github.sha }}` |
| `runner` | `${{ runner.os }}` |

### F. **Artifacts & Cache**

- **Upload Artifact** (`actions/upload-artifact@v3`)
- **Download Artifact** (`actions/download-artifact@v3`)
- **Cache** (`actions/cache@v3`) – store dependencies, build outputs.

### G. **Matrix with Custom Dependencies**

```yaml
strategy:
  matrix:
    include:
      - os: ubuntu-latest
        node-version: 14
        db: postgres
      - os: windows-latest
        node-version: 16
        db: mysql
```

Use `${{ matrix.db }}` in steps.

### H. **Using `needs` for Job Dependencies**

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    ...

  test:
    needs: build
    runs-on: ubuntu-latest
    ...
```

---

## Common Patterns

| Goal | Pattern |
|------|---------|
| **Linting** | Run ESLint in a separate job; cache `node_modules`. |
| **Build & Deploy** | Build on push to `main`, deploy if all tests pass. |
| **Multi‑environment CI** | Use matrix for OS/Node combos, then publish artifacts. |
| **Conditional Deployment** | `if: github.ref == 'refs/heads/main' && github.event_name == 'push'` |

---

## Useful Actions (quick links)

- `actions/checkout@v4`
- `actions/setup-node@v4`
- `actions/cache@v3`
- `actions/upload-artifact@v3`
- `actions/download-artifact@v3`
- `hashicorp/setup-terraform@v2` (for Terraform)
- `docker/build-push-action@v5` (Docker CI)

---

## Troubleshooting Tips

| Issue | Fix |
|-------|-----|
| “Missing permission for GITHUB_TOKEN” | Add `permissions:` to workflow or job: `permissions: contents: read, packages: write`. |
| Cache miss | Ensure key changes on dependency updates; use hash of lockfile. |
| Matrix job fails only on one OS | Check OS‑specific dependencies or paths (`${{ runner.os }}`). |

---

## Quick Commands

```bash
# Run all workflows locally (requires act)
act -j build,test,deploy

# List available actions
gh action list

# Inspect workflow logs
gh run view --log
```




---

# Anchors (`&`) & Aliases (`*`) Cheat Sheet

| Purpose | Where to put it | How to reference |
|---------|----------------|-----------------|
| **Reusable block** | `default: &my_block` (or any key you choose) | `${{ <context>.<name> }}` or `<<: *my_block` |

---

## Example: Repeated Env Block

```yaml
env:
  GLOBAL_URL: https://api.example.com

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Print URL
        run: echo "${{ env.GLOBAL_URL }}"
```

---

## Anchoring a *Complex* Section

```yaml
# Anchor once
default: &node_install
  run: npm ci

# Reuse it in two separate steps
steps:
  - name: Install deps (step 1)
    <<: *node_install

  - name: Run tests (step 2)
    <<: *node_install
```

---

## Anchors with **Inputs/Outputs**

```yaml
default: &set_tag
  run: echo "TAG=v${{ github.run_number }}" >> $GITHUB_OUTPUT

steps:
  - id: tag
    <<: *set_tag
  - name: Use TAG
    env:
      TAG: "${{ steps.tag.outputs.TAG }}"
```

---

## When to Use Anchors vs. `env`

| Scenario | Why anchor is useful |
|----------|---------------------|
| **Same matrix config** in multiple jobs | Define the whole matrix once and alias it (`<<: *matrix_config`). |
| **Long scripts** (Docker, Terraform) | Keep the script body anchored so you edit one line. |

---

### 📌 Quick YAML Template

```yaml
name: Build & Deploy

on:
  push:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  APP_NAME: myapp

# ── Anchor for Docker build -----------------
default: &docker_build
  run: |
    docker build -t ${{ env.REGISTRY }}/${{ env.APP_NAME }}:${{ github.sha }} .
    echo "IMAGE=${{ env.REGISTRY }}/${{ env.APP_NAME }}:${{ github.sha }}" >> $GITHUB_OUTPUT

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - <<: *docker_build   # pulls the whole block
```


---


## 📚 Quick‑Reference: Common `uses:` Actions for **Java Spring Boot** & **Python** Projects

URL: https://github.com/marketplace?type=actions

| # | Category | Action (repo) | Typical Use‑Case | Example Snippet |
|---|----------|----------------|------------------|-----------------|
| **Core** | |  |  |  |
| 1 | Checkout code | `actions/checkout@v4` | Pull your repo into the runner. | ```yaml- uses: actions/checkout@v4\n``` |
| 2 | Set up JDK (Java) | `actions/setup-java@v5` | Install a specific Java version and add Maven/Gradle to PATH. | ```yaml\n- name: Setup JDK\n  uses: actions/setup-java@v5\n  with:\n    distribution: 'temurin'\n    java-version: '21'\n``` |
| 3 | Set up Python | `actions/setup-python@v5` | Install a specific Python interpreter. | ```yaml\n- name: Setup Python\n  uses: actions/setup-python@v5\n  with:\n    python-version: '3.12'\n``` |
| **Build / Compile** | |  |  |
| 4 | Maven build (Java) | `actions/cache@v3` + `mvn -B package --file pom.xml` | Cache `.m2` repository for faster builds. | ```yaml\n- uses: actions/cache@v3\n  with:\n    path: ~/.m2/repository\n    key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}\n``` |
| 5 | Gradle build (Java) | `gradle/wrapper-validation-action@v1` + `./gradlew build` | Validate wrapper, run Gradle. | ```yaml\n- uses: gradle/wrapper-validation-action@v1\n- name: Build with Gradle\n  run: ./gradlew build --no-daemon\n``` |
| 6 | Python package install | `pip install -r requirements.txt` (script) or `pip-tools` cache | Cache pip packages. | ```yaml\n- uses: actions/cache@v3\n  with:\n    path: ~/.cache/pip\n    key: ${{ runner.os }}-pip-${{ hashFiles('requirements*.txt') }}\n``` |
| **Testing** | |  |  |
| 7 | Maven Surefire (Java) | `mvn test` (script) or `actions/setup-java` + `maven-test-action@v1` | Run unit tests. | ```yaml\n- name: Test with Maven\n  run: mvn -B test\n``` |
| 8 | Gradle Test | `./gradlew test` | Same for Gradle. | ```yaml\n- name: Test with Gradle\n  run: ./gradlew test --no-daemon\n``` |
| 9 | PyTest (Python) | `pytest` (script) or `actions/setup-python` + `pytest-action@v2` | Run Python tests. | ```yaml\n- name: Run PyTest\n  run: pytest -q\n``` |
| 10 | Codecov | `codecov/codecov-action@v4` | Upload coverage reports to Codecov. | ```yaml\n- uses: codecov/codecoord-action@v4\n  with:\n    files: ./coverage.xml\n``` |
| **Linting / Static Analysis** | |  |  |
| 11 | Checkstyle (Java) | `checkstyle/ checkstyle-action@v1` | Lint Java source. | ```yaml\n- uses: checkstyle/checkstyle-action@v1\n  with:\n    version: '10.12'\n``` |
| 12 | SpotBugs (Java) | `spotbugs/spotbugs-action@v2` | Find bugs in bytecode. | ```yaml\n- uses: spotbugs/spotbugs-action@v2\n  with:\n    toolVersion: '4.7.6'\n``` |
| 13 | Flake8 / Black (Python) | `psf/black@stable` & `pycqa/flake8@v6` | Format & lint Python code. | ```yaml\n- uses: psf/black@stable\n- uses: pycqa/flake8@v6\n``` |
| **Security** | |  |  |
| 14 | Trivy (Vulnerability Scan) | `aquasecurity/trivy-action@v0.1` | Scan Docker images or repository for CVEs. | ```yaml\n- uses: aquasecurity/trivy-action@v0.1\n  with:\n    image-ref: myapp:${{ github.sha }}\n``` |
| 15 | Dependabot Alerts (GitHub native) | `github/dependabot-alerts-action@v2` | Generate SARIF report for GitHub Security tab. | ```yaml\n- uses: github/dependabot-alerts-action@v2\n  with:\n    token: ${{ secrets.GITHUB_TOKEN }}\n``` |
| **Packaging / Release** | |  |  |
| 16 | Maven Publish | `mvn deploy` (script) or `gradle publish` | Deploy artifacts to Nexus/Artifactory. | ```yaml\n- name: Deploy Maven artifact\n  run: mvn -B deploy --settings .github/maven-settings.xml\n``` |
| 17 | Gradle Publishing | `./gradlew publish` | Publish a Gradle library. | ```yaml\n- name: Publish Gradle library\n  run: ./gradlew publish --no-daemon\n``` |
| 18 | Python Wheels | `pypa/build@v1` + `twine` | Build & upload Python wheel to PyPI. | ```yaml\n- uses: pypa/build@v1\n- name: Publish to PyPI\n  run: twine upload dist/*\n``` |
| **Docker** | |  |  |
| 19 | Docker Buildx | `docker/setup-buildx-action@v3` | Build multi‑arch images. | ```yaml\n- uses: docker/setup-buildx-action@v3\n- name: Build image\n  run: |\n    docker buildx build \\\n      --platform linux/amd64,linux/arm64 \\\n      -t ghcr.io/${{ github.repository }}:${{ github.sha }} \\\n      --push .\n``` |
| **CI/CD Orchestration** | |  |  |
| 20 | Matrix Strategy (built‑in) | N/A | Run jobs across OS/Java/Python combos. | ```yaml\nstrategy:\n  matrix:\n    os: [ubuntu-latest, windows-2022]\n    java-version: ['17', '21']\n``` |
| **Artifacts** | |  |  |
| 21 | Upload Artifacts | `actions/upload-artifact@v4` | Store test reports, logs, or binaries. | ```yaml\n- uses: actions/upload-artifact@v4\n  with:\n    name: test-reports\n    path: target/surefire-reports/**\n``` |
| 22 | Download Artifacts | `actions/download-artifact@v4` | Retrieve artifacts from previous jobs. | ```yaml\n- uses: actions/download-artifact@v4\n  with:\n    name: test-reports\n``` |

---

### 🚀 How to Pick the Right Action

1. **Language & Build Tool**  
   * Java → `setup-java`, Maven/Gradle actions, cache `.m2` or Gradle wrapper.  
   * Python → `setup-python`, pip‑cache, `build` for wheels.

2. **Task**  
   * Build → `mvn package` / `./gradlew build` / `pip install`.  
   * Test → unit tests (Maven/Gradle/PyTest).  
   * Lint → Checkstyle/SpotBugs for Java, Black/Flake8 for Python.  
   * Publish → Maven Deploy / Gradle publish / Twine upload.

3. **Environment**  
   * Docker → `setup-buildx` + `docker buildx`.  
   * Release artifacts → `upload-artifact`.

4. **Security & Compliance**  
   * Scan dependencies (Dependabot, Trivy).  
   * Generate SARIF for GitHub Security tab.

---

### 📋 Sample Full Workflow for a Spring Boot App

```yaml
name: CI / CD

on:
  push:
    branches: [ main ]

jobs:
  build-test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        java-version: ['17', '21']

    steps:
      - uses: actions/checkout@v4

      # ---- Java Setup ----
      - name: Setup JDK ${{ matrix.java-version }}
        uses: actions/setup-java@v5
        with:
          distribution: temurin
          java-version: ${{ matrix.java-version }}

      # ---- Cache Maven ----
      - uses: actions/cache@v3
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: |
            ${{ runner.os }}-maven-

      # ---- Build & Test ----
      - name: Build & test
        run: mvn -B clean package --file pom.xml

      # ---- Upload test reports ----
      - uses: actions/upload-artifact@v4
        with:
          name: surefire-reports-${{ matrix.java-version }}
          path: target/surefire-reports/**

  publish-docker:
    needs: build-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && success()
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - name: Build & push Docker image
        run: |
          docker buildx build \
            --platform linux/amd64,linux/arm64 \
            -t ghcr.io/${{ github.repository }}:${{ github.sha }} \
            --push .
```
