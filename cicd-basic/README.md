# CI/CD Basic

This course teaches CI/CD from first principles to production-grade pipelines using GitLab self-managed. You will install and configure a real GitLab server, write .gitlab-ci.yml pipelines, automate builds and tests, deploy to Kubernetes, manage secrets, and integrate with the GitOps model.

## Curriculum

<!-- 8 sections · 27 themes -->

### CI/CD Fundamentals and GitLab Installation

Introduces CI/CD concepts and the software delivery lifecycle, then walks through installing GitLab CE on Ubuntu, configuring the instance, and setting up GitLab Runner.

- [CI/CD Fundamentals](./cicd-fundamentals-installation/cicd-fundamentals.md)
- [Installing GitLab CE on Ubuntu](./cicd-fundamentals-installation/cicd-gitlab-installation.md)
- [Configuring GitLab](./cicd-fundamentals-installation/cicd-gitlab-configuration.md)
- [GitLab Runner — Installation and Registration](./cicd-fundamentals-installation/cicd-gitlab-runner.md)

### GitLab CI/CD Basics

Covers the GitLab CI/CD execution model, the structure of .gitlab-ci.yml, pipeline triggers, predefined variables, and reading pipeline status.

- [GitLab CI/CD Architecture](./cicd-gitlab-basics/cicd-gitlab-architecture.md)
- [Writing .gitlab-ci.yml](./cicd-gitlab-basics/cicd-gitlab-ci-yml.md)
- [Pipeline Triggers and Variables](./cicd-gitlab-basics/cicd-triggers-variables.md)

### Build and Test Automation

Covers building Docker images and application artifacts in CI, running unit tests with reports, collecting code coverage, linting, and parallelising tests across jobs.

- [Building Application Artifacts](./cicd-build-test/cicd-building-artifacts.md)
- [Testing, Coverage, and Linting](./cicd-build-test/cicd-testing-coverage.md)
- [Test Parallelism and Failing Fast](./cicd-build-test/cicd-test-parallelism.md)

### Artifacts, Caching, and Registries

Covers passing files between pipeline jobs using artifacts, speeding up pipelines with dependency caching, and using GitLab's built-in Container and Package registries.

- [Job Artifacts](./cicd-artifacts-caching/cicd-job-artifacts.md)
- [Dependency Caching](./cicd-artifacts-caching/cicd-caching.md)
- [Container and Package Registries](./cicd-artifacts-caching/cicd-registries.md)

### Deployment Strategies and Environments

Covers the main deployment strategies, defining environments in GitLab CI, deploying to Kubernetes, manual approval gates, and rolling back to a previous deployment.

- [Deployment Strategies](./cicd-deployments-environments/cicd-deployment-strategies.md)
- [Environments and Kubernetes Deployments](./cicd-deployments-environments/cicd-environments-k8s.md)
- [Manual Approvals and Rollback](./cicd-deployments-environments/cicd-approvals-rollback.md)

### Secrets and Variables Management

Covers GitLab CI/CD variable types, scoping variables to environments and branches, integrating HashiCorp Vault, securing Docker builds, and best practices for secret hygiene.

- [CI/CD Variables — Types, Masking, and Scoping](./cicd-secrets-variables/cicd-variables.md)
- [HashiCorp Vault Integration](./cicd-secrets-variables/cicd-vault-integration.md)
- [Docker Build Secrets and Best Practices](./cicd-secrets-variables/cicd-docker-secrets.md)

### Advanced Pipeline Patterns

Covers reusing pipeline configuration with includes and extends, conditional job execution with rules, multi-project pipelines, dynamic pipeline generation, and matrix builds.

- [Includes and Templates — DRY Pipelines](./cicd-advanced-patterns/cicd-includes-templates.md)
- [Rules and Conditional Execution](./cicd-advanced-patterns/cicd-rules-conditions.md)
- [Multi-Project Pipelines](./cicd-advanced-patterns/cicd-multi-project-pipelines.md)
- [Dynamic Pipelines and Matrix Builds](./cicd-advanced-patterns/cicd-dynamic-matrix-pipelines.md)

### GitOps Integration

Covers the GitOps model applied to CI/CD, separating application and config repositories, CI pipeline updates to the config repo, ArgoCD sync, promotion gates, and Helm chart registry.

- [The GitOps Model with GitLab](./cicd-gitops-integration/cicd-gitops-model.md)
- [CI Pipeline Updates the Config Repo](./cicd-gitops-integration/cicd-ci-updates-config-repo.md)
- [Notifications and Promotion Gates](./cicd-gitops-integration/cicd-notifications-promotion.md)
- [Helm Chart Registry — Write, Package, and Deploy via ArgoCD](./cicd-gitops-integration/cicd-helm-chart-registry.md)
