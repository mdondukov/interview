# 03. CI/CD

[← Назад к списку тем](README.md)

---

## CI/CD Concepts

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CI/CD Pipeline                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Continuous Integration (CI)                                        │
│  ─────────────────────────────                                      │
│  Code → Build → Test → Artifact                                     │
│                                                                     │
│  - Merge code frequently                                            │
│  - Automated builds on every commit                                 │
│  - Automated tests                                                  │
│  - Fast feedback                                                    │
│                                                                     │
│  Continuous Delivery (CD)                                           │
│  ──────────────────────────                                         │
│  Artifact → Staging → Production (manual approval)                  │
│                                                                     │
│  - Automated deployment to staging                                  │
│  - Manual approval for production                                   │
│  - Always deployable                                                │
│                                                                     │
│  Continuous Deployment                                              │
│  ────────────────────                                               │
│  Artifact → Staging → Production (automatic)                        │
│                                                                     │
│  - Fully automated pipeline                                         │
│  - Every passing commit goes to production                          │
│  - Requires high test confidence                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Pipeline Stages

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Typical Pipeline                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Source                                                          │
│     - Trigger: push, PR, schedule, manual                           │
│     - Checkout code                                                 │
│                                                                     │
│  2. Build                                                           │
│     - Install dependencies                                          │
│     - Compile code                                                  │
│     - Build artifacts (Docker image, binary)                        │
│                                                                     │
│  3. Test                                                            │
│     - Unit tests                                                    │
│     - Integration tests                                             │
│     - Security scans                                                │
│     - Linting                                                       │
│                                                                     │
│  4. Publish                                                         │
│     - Push to artifact registry                                     │
│     - Tag artifacts                                                 │
│                                                                     │
│  5. Deploy                                                          │
│     - Deploy to staging                                             │
│     - Run smoke tests                                               │
│     - Deploy to production                                          │
│     - Health checks                                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v4
        with:
          go-version: '1.21'

      - name: Cache dependencies
        uses: actions/cache@v3
        with:
          path: ~/go/pkg/mod
          key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}

      - name: Run tests
        run: go test -v -race -coverprofile=coverage.out ./...

      - name: Upload coverage
        uses: codecov/codecov-action@v3

  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: golangci-lint
        uses: golangci/golangci-lint-action@v3

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'

  build:
    needs: [test, lint, security]
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest

  deploy-staging:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Deploy to staging
        run: |
          # Deploy using kubectl, ArgoCD, etc.
          echo "Deploying to staging..."

  deploy-production:
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production  # Requires approval
    steps:
      - name: Deploy to production
        run: |
          echo "Deploying to production..."
```

---

## GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - deploy

variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

test:
  stage: test
  image: golang:1.21
  script:
    - go test -v -race ./...
  cache:
    paths:
      - /go/pkg/mod

lint:
  stage: test
  image: golangci/golangci-lint:latest
  script:
    - golangci-lint run

build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $DOCKER_IMAGE .
    - docker push $DOCKER_IMAGE
  only:
    - main

deploy_staging:
  stage: deploy
  environment:
    name: staging
  script:
    - kubectl set image deployment/app app=$DOCKER_IMAGE
  only:
    - main

deploy_production:
  stage: deploy
  environment:
    name: production
  script:
    - kubectl set image deployment/app app=$DOCKER_IMAGE
  when: manual
  only:
    - main
```

---

## Deployment Strategies

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Deployment Strategies                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ROLLING UPDATE                                                     │
│  ───────────────                                                    │
│  Old:  ████████████          Traffic gradually shifts               │
│  New:  ░░░░░░░░░░░░                                                 │
│        ↓                                                            │
│  Old:  ████████░░░░                                                 │
│  New:  ░░░░████████                                                 │
│                                                                     │
│  + Zero downtime                                                    │
│  + Gradual rollout                                                  │
│  - Slow rollback                                                    │
│  - Two versions running simultaneously                              │
│                                                                     │
│  BLUE-GREEN                                                         │
│  ───────────                                                        │
│  Blue (current):  ████████████ ← Traffic                            │
│  Green (new):     ████████████                                      │
│        ↓ Switch                                                     │
│  Blue (old):      ████████████                                      │
│  Green (current): ████████████ ← Traffic                            │
│                                                                     │
│  + Instant rollback                                                 │
│  + Full testing before switch                                       │
│  - Double resources                                                 │
│  - Database migrations tricky                                       │
│                                                                     │
│  CANARY                                                             │
│  ───────                                                            │
│  Stable:  ████████████████████ (90% traffic)                        │
│  Canary:  ████ (10% traffic)                                        │
│                                                                     │
│  + Test with real traffic                                           │
│  + Gradual risk                                                     │
│  + Quick rollback                                                   │
│  - More complex routing                                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Artifact Management

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Artifact Types                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Container Images                                                   │
│  - Docker Hub, ECR, GCR, ACR, Harbor                                │
│  - Tag: git sha, semver, latest                                     │
│                                                                     │
│  Language Packages                                                  │
│  - npm (Node), PyPI (Python), Maven (Java)                          │
│  - Private: Artifactory, Nexus                                      │
│                                                                     │
│  Binaries                                                           │
│  - GitHub Releases                                                  │
│  - S3, GCS                                                          │
│                                                                     │
│  Helm Charts                                                        │
│  - ChartMuseum, Harbor                                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Versioning Strategy

```
# Semantic Versioning
MAJOR.MINOR.PATCH (1.2.3)
- MAJOR: Breaking changes
- MINOR: New features (backward compatible)
- PATCH: Bug fixes

# Git SHA for traceability
image:abc123def

# Combined
image:1.2.3-abc123def

# Branch/environment tags
image:main-abc123def
image:staging
image:production
```

---

## Pipeline Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Pipeline Best Practices                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Fast feedback                                                   │
│     - Run fast tests first                                          │
│     - Fail fast on obvious issues                                   │
│     - Parallel jobs where possible                                  │
│                                                                     │
│  2. Reproducible builds                                             │
│     - Pin dependency versions                                       │
│     - Use deterministic builds                                      │
│     - Same artifact for all environments                            │
│                                                                     │
│  3. Security                                                        │
│     - Secrets in secret managers, not code                          │
│     - Scan dependencies for vulnerabilities                         │
│     - Scan containers for vulnerabilities                           │
│     - Sign artifacts                                                │
│                                                                     │
│  4. Traceability                                                    │
│     - Tag artifacts with git sha                                    │
│     - Keep build logs                                               │
│     - Track what's deployed where                                   │
│                                                                     │
│  5. Environment parity                                              │
│     - Same image in all environments                                │
│     - Environment-specific config via env vars                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## GitOps

```
┌─────────────────────────────────────────────────────────────────────┐
│                    GitOps Model                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Application Repo          Config Repo          Cluster             │
│  ┌─────────────┐          ┌─────────────┐     ┌─────────────┐      │
│  │   Code      │          │   K8s       │     │   ArgoCD    │      │
│  │   Changes   │ ────────▶│   Manifests │◀────│   Watches   │      │
│  └─────────────┘          └─────────────┘     └──────┬──────┘      │
│        │                         │                   │              │
│        │ CI builds               │                   │ Sync         │
│        │ image                   │                   ▼              │
│        ▼                         │            ┌─────────────┐       │
│  ┌─────────────┐                │            │   Cluster    │       │
│  │  Registry   │                │            │   State      │       │
│  └─────────────┘                │            └─────────────┘       │
│                                  │                                  │
│  Principles:                                                        │
│  - Git is single source of truth                                    │
│  - Declarative desired state                                        │
│  - Automated sync                                                   │
│  - Pull-based (more secure)                                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### ArgoCD Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/config-repo
    targetRevision: HEAD
    path: apps/myapp
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## Feature Flags

```go
// Using feature flags for safe deployments
type FeatureFlags struct {
    client *launchdarkly.LDClient
}

func (f *FeatureFlags) IsEnabled(flag string, user User) bool {
    return f.client.BoolVariation(flag, user.ToLDUser(), false)
}

// In application code
if featureFlags.IsEnabled("new-checkout-flow", currentUser) {
    return newCheckoutHandler(r)
}
return oldCheckoutHandler(r)
```

```
Benefits:
- Decouple deployment from release
- Gradual rollout (10% → 50% → 100%)
- Quick disable without redeploy
- A/B testing
```

---

## На интервью

### Типичные вопросы

1. **CI vs CD?**
   - CI: Automated build/test on every commit
   - CD: Automated deployment to staging/prod
   - Continuous Deployment: Fully automated to prod

2. **Rolling vs Blue-Green vs Canary?**
   - Rolling: Gradual replacement, zero downtime
   - Blue-Green: Two identical environments, instant switch
   - Canary: Small percentage first, gradual increase

3. **What makes a good pipeline?**
   - Fast feedback
   - Reproducible builds
   - Security scanning
   - Traceability
   - Same artifact across environments

4. **How do you handle secrets?**
   - Never in code/config
   - Secret managers (Vault, AWS Secrets Manager)
   - Injected at runtime
   - Rotated regularly

5. **How do you handle database migrations?**
   - Forward-compatible changes
   - Run migrations before deployment
   - Backward-compatible rollbacks
   - Blue-green: more complex (need to handle both)

6. **GitOps benefits?**
   - Git as single source of truth
   - Audit trail
   - Easy rollback (git revert)
   - Pull-based (more secure than push)

---

[← Назад к списку тем](README.md)
