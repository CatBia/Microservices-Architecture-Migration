# Reusable Workflows Implementation

## Overview

This project implements **GitHub Actions Reusable Workflows** to follow DRY (Don't Repeat Yourself) principles and ensure consistency across all microservices.

## Architecture

```
.github/workflows/
├── ci.yml                    # Main CI pipeline (uses reusable workflows)
├── cd.yml                    # Main CD pipeline (uses reusable workflows)
├── pages.yml                  # GitHub Pages deployment
└── reusable/                 # Reusable workflow library
    ├── test-nodejs.yml       # Node.js service testing
    ├── test-go.yml           # Go service testing
    ├── build-docker.yml      # Docker image build & push
    ├── deploy-kubernetes.yml # Kubernetes deployment
    ├── security-scan.yml     # Security scanning
    └── README.md            # Detailed documentation
```

## How It Works

### 1. Reusable Workflows (`.github/workflows/reusable/`)

These are standalone workflows that can be called from other workflows using the `workflow_call` trigger.

**Example:**
```yaml
# .github/workflows/reusable/test-nodejs.yml
name: Test Node.js Service

on:
  workflow_call:
    inputs:
      service_path:
        required: true
        type: string
    # ... more inputs
```

### 2. Calling Workflows (`.github/workflows/ci.yml`, `cd.yml`)

Main workflows call reusable workflows using the `uses` keyword.

**Example:**
```yaml
# .github/workflows/ci.yml
jobs:
  test-api-gateway:
    uses: ./.github/workflows/reusable/test-nodejs.yml
    with:
      service_path: 'services/api-gateway'
      service_name: 'api-gateway'
```

## Benefits

### ✅ Consistency
All services use identical testing, building, and deployment processes.

### ✅ Maintainability
Update a workflow once, and all services benefit from the change.

### ✅ Scalability
Add new services without duplicating workflow code.

### ✅ DRY Principle
Write workflow logic once, reuse everywhere.

### ✅ Version Control
All workflow changes are tracked in git with full history.

## Workflow Flow

```
┌─────────────────────────────────────────┐
│  Developer pushes code / PR created     │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│  CI Pipeline (ci.yml)                   │
│  ├── test-nodejs.yml (API Gateway)       │
│  ├── test-go.yml (User Service)         │
│  ├── test-go.yml (Product Service)      │
│  ├── security-scan.yml                  │
│  └── build-docker.yml (all services)    │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│  CD Pipeline (cd.yml)                    │
│  └── deploy-kubernetes.yml              │
│      ├── dev environment                 │
│      ├── staging environment            │
│      └── production environment         │
└─────────────────────────────────────────┘
```

## Adding a New Service

To add a new service to the CI/CD pipeline:

1. **Add test job** (if Node.js):
```yaml
test-new-service:
  uses: ./.github/workflows/reusable/test-nodejs.yml
  with:
    service_path: 'services/new-service'
    service_name: 'new-service'
```

2. **Add build job**:
```yaml
build-new-service:
  needs: [test-new-service]
  uses: ./.github/workflows/reusable/build-docker.yml
  with:
    service_path: services/new-service
    service_name: new-service
```

3. **Add deployment** (if needed):
```yaml
deploy-new-service:
  uses: ./.github/workflows/reusable/deploy-kubernetes.yml
  with:
    environment: dev
    service_name: new-service
```

That's it! No need to duplicate workflow code.

## Artifacts in Workflows

**Artifacts** are files produced during workflow runs that are preserved for later use. They are essential for:

- 🔍 **Debugging**: Download test results, logs, and error outputs to investigate failures
- 📊 **Test Results & Coverage**: Store test outputs and coverage reports for analysis
- 📦 **Build Outputs**: Preserve compiled binaries, Docker image metadata, and build logs
- 🔄 **Cross-Job Communication**: Share files between jobs in the same workflow
- 📋 **Compliance & Auditing**: Maintain records of builds and deployments

**Best Practices for Artifacts:**
- Use `if: always()` to upload artifacts even when jobs fail
- Set appropriate `retention-days` based on artifact importance
- Use descriptive artifact names with context (service name, SHA, etc.)
- Handle missing files gracefully with `if-no-files-found: warn`

📖 **See [`.github/workflows/ARTIFACTS.md`](ARTIFACTS.md) for comprehensive artifact documentation.**

## Best Practices

1. **Keep workflows focused**: Each reusable workflow should have a single responsibility
2. **Use descriptive inputs**: Make inputs self-documenting
3. **Provide defaults**: Use sensible defaults for optional inputs
4. **Document everything**: Update README.md when adding new workflows
5. **Test locally**: Use `act` or GitHub Actions locally before committing
6. **Version control**: All workflow changes go through PR review
7. **Upload artifacts**: Preserve test results, build outputs, and deployment information for debugging and compliance

## Migration from Monolithic Workflows

If you have existing monolithic workflows, migration is straightforward:

**Before:**
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Node.js
        uses: actions/setup-node@v4
      # ... many more steps
```

**After:**
```yaml
jobs:
  test:
    uses: ./.github/workflows/reusable/test-nodejs.yml
    with:
      service_path: 'services/my-service'
      service_name: 'my-service'
```

## Resources

- [GitHub Actions Reusable Workflows Documentation](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [Project Reusable Workflows README](.github/workflows/reusable/README.md)
- [Build Guide - CI/CD Section](../../docs/BUILD_GUIDE.md#cicd-pipeline)

