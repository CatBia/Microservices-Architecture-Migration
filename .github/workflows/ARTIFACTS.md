# GitHub Actions Artifacts Guide

## What are Artifacts?

**Artifacts** are files or collections of files produced during a workflow run that you want to preserve after the job completes. They are stored in GitHub's artifact storage and can be downloaded, shared between jobs, or used for debugging and analysis.

## Why Use Artifacts?

### 1. **Debugging & Troubleshooting**
- Preserve test results, logs, and error outputs for later analysis
- Download artifacts to investigate failed builds locally
- Share artifacts with team members for collaborative debugging

### 2. **Test Results & Coverage Reports**
- Store test results in standard formats (JUnit XML, etc.)
- Preserve code coverage reports for analysis
- Track test trends over time

### 3. **Build Outputs**
- Save compiled binaries, Docker images metadata
- Preserve build logs for audit trails
- Store deployment manifests and configurations

### 4. **Cross-Job Communication**
- Pass files between jobs in the same workflow
- Share build outputs from one job to another
- Enable downstream jobs to use artifacts from upstream jobs

### 5. **Compliance & Auditing**
- Maintain records of what was built and deployed
- Keep deployment manifests for rollback purposes
- Store security scan results for compliance

## Common Artifact Types in CI/CD

### Test Artifacts
```yaml
- name: Upload test results
  uses: actions/upload-artifact@v3
  if: always()
  with:
    name: test-results
    path: test-results/
    retention-days: 30
```

**Contains:**
- Test output files (JUnit XML, JSON)
- Test screenshots/videos
- Test logs

### Coverage Reports
```yaml
- name: Upload coverage report
  uses: actions/upload-artifact@v3
  if: always()
  with:
    name: coverage-report
    path: coverage/
    retention-days: 30
```

**Contains:**
- Code coverage HTML reports
- Coverage data files (LCOV, Cobertura)
- Coverage summaries

### Build Artifacts
```yaml
- name: Upload build artifacts
  uses: actions/upload-artifact@v3
  if: always()
  with:
    name: build-outputs
    path: dist/
    retention-days: 7
```

**Contains:**
- Compiled binaries
- Docker image metadata
- Build logs
- Package files

### Deployment Artifacts
```yaml
- name: Upload deployment manifests
  uses: actions/upload-artifact@v3
  if: always()
  with:
    name: deployment-manifests
    path: kubernetes/overlays/production/
    retention-days: 90
```

**Contains:**
- Kubernetes manifests
- Deployment configurations
- Environment-specific settings

### Security Scan Results
```yaml
- name: Upload security scan results
  uses: actions/upload-artifact@v3
  if: always()
  with:
    name: security-scan-results
    path: security-reports/
    retention-days: 90
```

**Contains:**
- Vulnerability reports
- SAST/DAST scan results
- Dependency audit reports

## Best Practices

### 1. Use Conditional Uploads
Always use `if: always()` to ensure artifacts are uploaded even if the job fails:

```yaml
- name: Upload logs
  uses: actions/upload-artifact@v3
  if: always()  # Upload even if job fails
  with:
    name: logs
    path: logs/
```

### 2. Set Appropriate Retention Periods
Balance storage costs with retention needs:

```yaml
retention-days: 30  # Short-term: test results
retention-days: 90  # Medium-term: build artifacts
retention-days: 365 # Long-term: compliance/audit
```

### 3. Use Descriptive Names
Make artifact names descriptive and include context:

```yaml
# Good
name: api-gateway-test-results-${{ github.run_number }}

# Better
name: ${{ inputs.service_name }}-test-results-${{ github.sha }}
```

### 4. Handle Missing Files Gracefully
Use `if-no-files-found` to avoid failures when files don't exist:

```yaml
with:
  name: coverage
  path: coverage/
  if-no-files-found: warn  # or 'ignore' or 'error'
```

### 5. Compress Large Artifacts
For large files, consider compression:

```yaml
- name: Compress logs
  run: tar -czf logs.tar.gz logs/

- name: Upload compressed logs
  uses: actions/upload-artifact@v3
  with:
    name: logs-compressed
    path: logs.tar.gz
```

## Artifact Workflow Patterns

### Pattern 1: Test → Build → Deploy

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Run tests
        run: npm test
      
      - name: Upload test results
        uses: actions/upload-artifact@v3
        with:
          name: test-results
          path: test-results/

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Download test results
        uses: actions/download-artifact@v3
        with:
          name: test-results
      
      - name: Build
        run: npm run build
      
      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: build-outputs
          path: dist/

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Download build artifacts
        uses: actions/download-artifact@v3
        with:
          name: build-outputs
      
      - name: Deploy
        run: ./deploy.sh
```

### Pattern 2: Parallel Jobs with Shared Artifacts

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Build
        run: npm run build
      
      - name: Upload build
        uses: actions/upload-artifact@v3
        with:
          name: build-outputs
          path: dist/

  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Download build
        uses: actions/download-artifact@v3
        with:
          name: build-outputs
      
      - name: Test
        run: npm test

  security-scan:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Download build
        uses: actions/download-artifact@v3
        with:
          name: build-outputs
      
      - name: Scan
        run: npm audit
```

## Artifact Storage & Limits

### Storage Limits
- **Free accounts**: 500 MB storage, 1 GB transfer/month
- **Pro accounts**: 1 GB storage, 2 GB transfer/month
- **Team accounts**: 2 GB storage, 4 GB transfer/month
- **Enterprise**: 50 GB storage, 50 GB transfer/month

### Best Practices for Storage
1. **Clean up old artifacts**: Set appropriate `retention-days`
2. **Compress large files**: Use compression for logs and reports
3. **Selective uploads**: Only upload what's necessary
4. **Use external storage**: For very large artifacts, use S3, Azure Blob, etc.

## Downloading Artifacts

### Via GitHub UI
1. Go to the workflow run
2. Scroll to "Artifacts" section
3. Click to download

### Via GitHub CLI
```bash
gh run download <run-id>
gh run download <run-id> --name <artifact-name>
```

### Via API
```bash
curl -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/OWNER/REPO/actions/artifacts
```

## Example: Complete CI/CD with Artifacts

```yaml
name: CI/CD Pipeline with Artifacts

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test
        continue-on-error: true
      
      - name: Generate coverage
        run: npm run test:coverage
        continue-on-error: true
      
      - name: Upload test results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: test-results-${{ github.sha }}
          path: |
            test-results/
            coverage/
          retention-days: 30
          if-no-files-found: warn

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build
        run: npm run build
      
      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: build-${{ github.sha }}
          path: dist/
          retention-days: 7

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Security scan
        run: npm audit --json > security-report.json
        continue-on-error: true
      
      - name: Upload security report
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: security-report-${{ github.sha }}
          path: security-report.json
          retention-days: 90

  deploy:
    needs: [build, security]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Download build artifacts
        uses: actions/download-artifact@v3
        with:
          name: build-${{ github.sha }}
          path: dist/
      
      - name: Deploy
        run: ./deploy.sh
      
      - name: Save deployment manifest
        run: |
          echo "${{ github.sha }}" > deployment-manifest.txt
          echo "${{ github.ref }}" >> deployment-manifest.txt
      
      - name: Upload deployment manifest
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: deployment-manifest-${{ github.sha }}
          path: deployment-manifest.txt
          retention-days: 365
```

## Troubleshooting

### Artifact Not Found
- Check if the artifact was actually created
- Verify the path is correct
- Ensure `if: always()` is used if job might fail

### Artifact Too Large
- Compress files before uploading
- Use external storage (S3, etc.)
- Split into multiple smaller artifacts

### Download Fails
- Check artifact name matches exactly
- Verify artifact exists in the workflow run
- Ensure download happens in a job that depends on the upload job

## Resources

- [GitHub Actions Artifacts Documentation](https://docs.github.com/en/actions/using-workflows/storing-workflow-data-as-artifacts)
- [upload-artifact Action](https://github.com/actions/upload-artifact)
- [download-artifact Action](https://github.com/actions/download-artifact)
- [Artifact Storage Limits](https://docs.github.com/en/billing/managing-billing-for-github-actions/about-billing-for-github-actions#artifact-storage)

