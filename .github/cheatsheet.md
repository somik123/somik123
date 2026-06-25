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
