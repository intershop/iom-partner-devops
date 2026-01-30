# IOM Partner DevOps - AI Agent Instructions

## Project Overview

This repository provides centralized Azure DevOps CI/CD pipeline templates for Intershop Order Management (IOM) partner projects. It's a **template library**, not an application—consumer projects reference `ci-job-template.yml` via Azure Pipelines to standardize build, test, and deployment workflows.

## Architecture

### Core Components

- **`ci-job-template.yml`**: The main reusable pipeline template (794 lines) that orchestrates the entire CI/CD workflow
- **`azure-pipelines.yml`**: Example consumer pipeline demonstrating how partner projects use this template
- **README.rst**: Comprehensive integration guide for partner projects

### Pipeline Workflow (ci-job-template.yml)

The template executes a complete containerized testing cycle:

1. **Environment Setup** → validates parameters, checks out project, determines IOM version and JDK requirements, displays available memory
2. **Build Phase** → creates Docker images using Maven (`mvn clean package -Pdocker`) within Minikube's Docker daemon
3. **Deployment Phase** → installs IOM via Helm charts in local Minikube cluster with PostgreSQL, NGINX ingress, and replicas
4. **Diagnostics Capture** → collects comprehensive Kubernetes cluster status, pod descriptions, events, and container logs for troubleshooting
5. **Test Phase** → runs integration tests using Maven (`mvn verify`) with Failsafe against the deployed IOM instance
6. **Publication Phase** → pushes tested images to Azure Container Registry (ACR) only for protected branches matching `branchesForPublication` regex

## Critical Configuration Patterns

### Parameter Validation
All required parameters are validated at pipeline start. Empty values for `dockerRepoIOMServiceConnection`, `dockerRepoIOM`, `acrServiceConnection`, `acr`, or `artifactsFeed` cause immediate failure with clear error messages.

### Branch-Based Publication
```yaml
branchesForPublication: '^refs/heads/develop$\|^refs/heads/master$\|^refs/heads/main$\|^refs/heads/release/\|^refs/heads/hotfix/'
```
Only branches matching this regex publish images to ACR. Variable `IS_PRIVATE_BRANCH` controls conditional execution.

### Version Management
- **Unique SNAPSHOT tags**: When `uniqueSnapshotTag: True` and `IS_RELEASE_BUILD: false`, images get `-<commit-hash>` suffix
- **Release builds**: Detected when `PROJECT_VERSION` doesn't end with "SNAPSHOT"
- **IOM version**: Extracted from Maven property `platform.version` in project's pom.xml

### Resource Requirements
Minikube setup: `--cpus=max --memory=12000m` (increased from 8GB to 12GB for stability)  
The pipeline checks available memory with `free -h` before starting Minikube and after Helm installation.

Pod resources in values.yaml:
```yaml
resources:
  limits: {cpu: 1000m, memory: 3500Mi}
  requests: {cpu: 1000m, memory: 3500Mi}
jboss:
  javaOpts: "-XX:+UseContainerSupport -XX:MinRAMPercentage=60 -XX:MaxRAMPercentage=60"
postgres:
  resources:
    requests: {cpu: 1000m, memory: 3000Mi}  # Increased from 1000Mi
    limits: {cpu: 2000m, memory: 3000Mi}
```

## Developer Workflows

### Testing Changes to ci-job-template.yml
Partner projects cannot directly test template changes. Use `azure-pipelines.yml` as integration test:
- Defines 4 blueprint project instances (`iom-blueprint-project-develop`, `1-5-0`, `1-6-0`, `1-7-0`)
- Each blueprint tests template against different IOM versions via loop
- Runs nightly (`cron: "0 2 * * *"`) to detect environment drift

### Adding Pre/Post Hooks
Consumer projects can inject custom steps:
```yaml
preHookTemplate: custom-pre-steps.yml@self
postHookTemplate: custom-post-steps.yml@self
```
Hooks execute before/after main CI steps. Use `@self` to reference templates in consumer repo.

### Debugging Failed Pipelines
Pipeline publishes extensive diagnostics on failure:
- **Build Summary**: `config${{ parameters.id }}.md` shows all resolved variables
- **Helm Values**: `values${{ parameters.id }}.md` contains complete Helm configuration
- **Installation Diagnostics**: Automatic capture after Helm install includes:
  - Kubernetes cluster status (`kubectl get all`)
  - Pod descriptions with events and conditions
  - Namespace events sorted by timestamp
  - Container logs from all pods (last 100 lines per container)
  - PostgreSQL and NGINX ingress logs
- **K8s Status**: `kubernetes${{ parameters.id }}.md` shows pod states
- **Logs Artifact**: `iom-logs${{ parameters.id }}` includes pod logs, kubectl describe output, surefire output
- **Memory Monitoring**: `free -h` output before Minikube start and after Helm installation

## Key Conventions

### Minikube Docker Environment
All Docker builds/operations use Minikube's daemon via `eval $(minikube -p minikube docker-env)`. Images built locally never need registry pulls during deployment.

### Caching Strategy
Two pipeline caches optimize runtime:
- `minikube | "$(IOM_VERSION)" | "$(PROJECT_VERSION)"`: Stores Minikube VM data
- `mvn repo cache | "$(IOM_VERSION)" | "$(PROJECT_VERSION)"`: Stores Maven dependencies

### Job Naming with ID Parameter
The `id` parameter (must be `[a-zA-Z0-9_]` only) makes job names unique when template is called in loops: `CI${{ parameters.id }}`. It also suffixes artifacts and log files.

## External Dependencies

- **IOM Helm Charts**: `https://intershop.github.io/helm-charts` (version controlled via `IOM_HELM_VERSION`)
- **Azure Artifacts**: Maven authentication via `MavenAuthenticate@0` task connects to customer feed
- **Service Connections**: Requires Azure DevOps service connections for both Intershop registry (base images) and customer ACR (published images)
- **Variable Groups**: `iom-build-configuration` library provides `BUILD_AGENT_POOL`, registry connection names, and ACR paths

## Common Pitfalls

1. **Memory Requirements**: Build agents must have at least 12GB+ free memory for Minikube. Check `free -h` output if installation fails with "context deadline exceeded" errors
2. **JVM Heap Configuration**: IOM pods use 60% of container memory for heap (`-XX:MinRAMPercentage=60 -XX:MaxRAMPercentage=60`). Insufficient memory causes OOM kills
3. **JDK Version Detection**: Uses `xml_grep` on `//plugins/plugin/configuration/release` which may fail if Maven Compiler Plugin isn't structured as expected
4. **Image Tagging**: SNAPSHOT images without `uniqueSnapshotTag` will overwrite previous tags—use unique tags for reliable artifact tracking
5. **Test Data Import**: Pipeline waits `IMPORT_TESTDATA_TIMEOUT + 60` seconds after Helm install, but doesn't verify import success—pod restarts indicate failure
6. **LoadBalancer IPs**: Requires `minikube tunnel` running in background; IP extraction fails silently if tunnel isn't established
7. **Rate Limiter Errors**: "client rate limiter Wait returned an error" during Helm install usually indicates resource exhaustion—check pod events and container logs in diagnostics output

## Modifying the Template

When editing `ci-job-template.yml`:
- Test against multiple IOM versions using the blueprint pattern in `azure-pipelines.yml`
- Maintain backward compatibility—partner projects may not update immediately
- Document new parameters in README.rst with clear examples
- Consider impact on cache keys (changes to keys invalidate all cached data)
