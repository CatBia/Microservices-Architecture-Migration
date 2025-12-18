# Reusable Workflows

This directory contains reusable GitHub Actions workflows that can be called from other workflows in this repository or from external repositories.

## Overview

Reusable workflows promote DRY (Don't Repeat Yourself) principles by centralizing common CI/CD patterns. This makes it easier to maintain consistency across all services and reduces duplication.

## Available Workflows

### 1. `test-nodejs.yml` - Node.js Service Testing

Tests Node.js-based services with linting, unit tests, and coverage reporting.

**Inputs:**
- `service_path` (required): Path to the service directory
- `service_name` (required): Name of the service
- `node_version` (optional): Node.js version (default: '18')
- `NPM_TOKEN` (optional secret): NPM token for private packages

**Usage:**
```yaml
jobs:
  test-api-gateway:
    uses: ./.github/workflows/reusable/test-nodejs.yml
    with:
      service_path: 'services/api-gateway'
      service_name: 'api-gateway'
      node_version: '18'
    secrets:
      NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### 2. `test-go.yml` - Go Service Testing

Tests Go-based services with linting, unit tests, and coverage reporting.

**Inputs:**
- `service_path` (required): Path to the service directory
- `service_name` (required): Name of the service
- `go_version` (optional): Go version (default: '1.21')

**Usage:**
```yaml
jobs:
  test-user-service:
    uses: ./.github/workflows/reusable/test-go.yml
    with:
      service_path: 'services/user-service'
      service_name: 'user-service'
      go_version: '1.21'
```

### 3. `build-docker.yml` - Docker Image Build and Push

Builds and optionally pushes Docker images to a container registry.

**Inputs:**
- `service_path` (required): Path to the service directory
- `service_name` (required): Name of the service
- `dockerfile_path` (optional): Path to Dockerfile (default: 'Dockerfile')
- `image_registry` (optional): Docker registry URL (default: 'docker.io')
- `push_image` (optional): Whether to push the image (default: true)
- `DOCKER_USERNAME` (required secret): Docker registry username
- `DOCKER_PASSWORD` (required secret): Docker registry password

**Usage:**
```yaml
jobs:
  build:
    uses: ./.github/workflows/reusable/build-docker.yml
    with:
      service_path: 'services/api-gateway'
      service_name: 'api-gateway'
      push_image: true
    secrets:
      DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
      DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
```

### 4. `deploy-kubernetes.yml` - Kubernetes Deployment

Deploys services to Kubernetes clusters (EKS).

**Inputs:**
- `environment` (required): Deployment environment (dev, staging, production)
- `service_name` (required): Name of the service to deploy
- `namespace` (optional): Kubernetes namespace (default: 'movister')
- `image_tag` (optional): Docker image tag (default: 'latest')
- `k8s_manifest_path` (optional): Path to Kubernetes manifests (default: 'kubernetes/overlays')
- `aws_region` (optional): AWS region (default: 'us-east-1')
- `eks_cluster_name` (optional): EKS cluster name (default: 'movister-dev')
- `AWS_ACCESS_KEY_ID` (required secret): AWS access key ID
- `AWS_SECRET_ACCESS_KEY` (required secret): AWS secret access key

**Usage:**
```yaml
jobs:
  deploy:
    uses: ./.github/workflows/reusable/deploy-kubernetes.yml
    with:
      environment: dev
      service_name: 'api-gateway'
      namespace: movister
      image_tag: ${{ github.sha }}
      eks_cluster_name: movister-dev
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

### 5. `security-scan.yml` - Security Scanning

Performs security scanning using Trivy and CodeQL.

**Inputs:**
- `service_path` (required): Path to the service directory
- `service_name` (required): Name of the service
- `scan_type` (optional): Type of scan - 'all', 'code', 'dependencies', 'container' (default: 'all')
- `GITHUB_TOKEN` (required secret): GitHub token for security scanning

**Usage:**
```yaml
jobs:
  security-scan:
    uses: ./.github/workflows/reusable/security-scan.yml
    with:
      service_path: '.'
      service_name: 'movister-parallel-porter'
      scan_type: 'all'
    secrets:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Benefits of Reusable Workflows

1. **Consistency**: All services use the same testing, building, and deployment processes
2. **Maintainability**: Update workflows in one place, changes apply everywhere
3. **Reduced Duplication**: No need to copy-paste workflow code across services
4. **Version Control**: Workflow changes are tracked in git
5. **Reusability**: Can be used across multiple repositories

## Adding a New Reusable Workflow

1. Create a new YAML file in `.github/workflows/reusable/`
2. Use `workflow_call` trigger
3. Define inputs and secrets
4. Document in this README
5. Update main CI/CD workflows to use it

## Artifacts

All reusable workflows can be enhanced with artifact uploads to preserve test results, build outputs, and deployment information. This is essential for:

- **Debugging**: Download artifacts to investigate failed builds
- **Test Results**: Store test outputs and coverage reports
- **Build Outputs**: Preserve build metadata and logs
- **Cross-Job Communication**: Share files between jobs
- **Compliance**: Maintain audit trails

**Example: Adding artifacts to a workflow**

```yaml
- name: Upload test results
  uses: actions/upload-artifact@v3
  if: always()
  with:
    name: ${{ inputs.service_name }}-test-results
    path: test-results/
    retention-days: 30
```

📖 **See [`.github/workflows/ARTIFACTS.md`](../ARTIFACTS.md) for comprehensive artifact documentation.**

## Best Practices

- Keep workflows focused on a single responsibility
- Use descriptive input and secret names
- Provide sensible defaults for optional inputs
- Document all inputs and secrets
- Test workflows in development environment first
- Use matrix strategies for multiple services
- **Upload artifacts** for debugging and compliance

## Examples

See `.github/workflows/ci.yml` and `.github/workflows/cd.yml` for examples of how these reusable workflows are used.

