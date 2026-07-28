# Flux Operator Technical Reference

This document provides deep technical knowledge of how the Flux Operator works, including its architecture, reconciliation mechanics, and relationship to base Flux.

## What is the Flux Operator?

The Flux Operator is a **Kubernetes operator** that extends base Flux with additional capabilities. It's developed by ControlPlane.io (the same team behind Flux) and licensed under AGPL-3.0.

### Base Flux vs Flux Operator

| Component | What It Does | CRDs |
|-----------|--------------|------|
| **Base Flux** | GitOps reconciliation | GitRepository, Kustomization, HelmRelease, HelmRepository, etc. |
| **Flux Operator** | Manages Flux itself + adds features | FluxInstance, ResourceSet, ResourceSetInputProvider, FluxReport |

Base Flux is installed via `flux bootstrap`. The Flux Operator is an **add-on** installed via Helm that:
1. Can manage Flux installations declaratively (FluxInstance)
2. Provides dynamic resource generation (ResourceSet)
3. Adds reporting and monitoring (FluxReport)

**We use the Operator primarily for ResourceSet/ResourceSetInputProvider**, not for managing Flux itself.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Flux Operator Pod                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                        Operator Controller                          │    │
│  │                                                                     │    │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐   │    │
│  │  │ FluxInstance     │  │ ResourceSet      │  │ ResourceSetInput │   │    │
│  │  │ Reconciler       │  │ Reconciler       │  │ Provider         │   │    │
│  │  │                  │  │                  │  │ Reconciler       │   │    │
│  │  │ Manages Flux     │  │ Templates and    │  │                  │   │    │
│  │  │ installations    │  │ creates K8s      │  │ Fetches inputs   │   │    │
│  │  │                  │  │ resources        │  │ from external    │   │    │
│  │  │                  │  │                  │  │ sources          │   │    │
│  │  └──────────────────┘  └──────────────────┘  └──────────────────┘   │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                    watches/creates │ Kubernetes resources
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Kubernetes API Server                               │
│                                                                             │
│  Custom Resources (Flux Operator)     Native Resources (created by operator)│
│  ┌─────────────────────────────┐      ┌─────────────────────────────┐       │
│  │ ResourceSetInputProvider    │      │ GitRepository               │       │
│  │ ResourceSet                 │      │ Kustomization               │       │
│  │ FluxInstance                │      │ HelmRelease                 │       │
│  │ FluxReport                  │      │ Namespace, ConfigMap, etc.  │       │
│  └─────────────────────────────┘      └─────────────────────────────┘       │
└─────────────────────────────────────────────────────────────────────────────┘
```

## ResourceSetInputProvider Deep Dive

### Purpose

The ResourceSetInputProvider **fetches data from external sources** and exports it as a list of inputs for ResourceSets to consume. It abstracts away the complexity of polling external APIs.

### Supported Input Types

| Type | Source | Use Case |
|------|--------|----------|
| `GitHubBranch` | GitHub REST API | Deploy from feature branches |
| `GitHubPullRequest` | GitHub REST API | Preview environments for PRs |
| `GitLabBranch` | GitLab REST API | Feature branch deployments |
| `GitLabMergeRequest` | GitLab REST API | MR preview environments |
| `GitLabEnvironment` | GitLab REST API | Environment-based deployments |
| `AzureDevOpsBranch` | Azure DevOps API | Branch deployments |
| `GiteaBranch` | Gitea REST API | Self-hosted git branch deployments |

### How It Works (GitHubBranch)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ResourceSetInputProvider Controller                  │
│                                                                         │
│  1. Read spec from CR                                                   │
│     ┌─────────────────────────────────────────────┐                     │
│     │ spec:                                       │                     │
│     │   type: GitHubBranch                        │                     │
│     │   url: https://github.com/owner/repo        │                     │
│     │   filter:                                   │                     │
│     │     includeBranch: "^test/.*"               │                     │
│     └─────────────────────────────────────────────┘                     │
│                              │                                          │
│  2. Call GitHub API          │                                          │
│     GET /repos/owner/repo/branches                                      │
│                              │                                          │
│                              ▼                                          │
│     ┌─────────────────────────────────────────────┐                     │
│     │ GitHub Response:                            │                     │
│     │ [                                           │                     │
│     │   { "name": "main", "sha": "abc123" },      │                     │
│     │   { "name": "test/hello", "sha": "def456" },│                     │
│     │   { "name": "test/media", "sha": "ghi789" },│                     │
│     │   { "name": "feature/x", "sha": "jkl012" }  │                     │
│     │ ]                                           │                     │
│     └─────────────────────────────────────────────┘                     │
│                              │                                          │
│  3. Apply filter (regex)     │ includeBranch: "^test/.*"                │
│                              ▼                                          │
│     ┌─────────────────────────────────────────────┐                     │
│     │ Filtered:                                   │                     │
│     │ [                                           │                     │
│     │   { "name": "test/hello", "sha": "def456" },│                     │
│     │   { "name": "test/media", "sha": "ghi789" } │                     │
│     │ ]                                           │                     │
│     └─────────────────────────────────────────────┘                     │
│                              │                                          │
│  4. Transform to inputs      │ (add computed fields)                    │
│                              ▼                                          │
│     ┌─────────────────────────────────────────────┐                     │
│     │ status.exportedInputs:                      │                     │
│     │ - branch: "test/hello"                      │                     │
│     │   sha: "def456..."                          │                     │
│     │   id: "371459076"  # hash of branch name    │                     │
│     │ - branch: "test/media"                      │                     │
│     │   sha: "ghi789..."                          │                     │
│     │   id: "892734561"                           │                     │
│     └─────────────────────────────────────────────┘                     │
│                              │                                          │
│  5. Update CR status         │                                          │
│  6. Notify ResourceSets      │ (triggers their reconciliation)          │
└──────────────────────────────┼──────────────────────────────────────────┘
                               ▼
                    ResourceSet sees new inputs
```

### Exported Fields per Input Type

For `GitHubBranch`:
```yaml
status:
  exportedInputs:
    - branch: "test/hello"     # Full branch name
      sha: "def456abc..."      # Latest commit SHA (40 chars)
      id: "371459076"          # Numeric hash, safe for K8s names
```

The `id` field is crucial — it's a **stable numeric identifier** derived from the branch name, safe to use in Kubernetes resource names (which have strict naming rules).

### Reconciliation Interval

Controlled by annotation:
```yaml
metadata:
  annotations:
    fluxcd.controlplane.io/reconcileEvery: "5m"  # Poll interval
```

This is **independent of the Kustomization interval** — it controls how often the operator calls the GitHub API.

### Authentication

For public repos, no auth needed. For private repos or higher rate limits:

```yaml
spec:
  secretRef:
    name: github-token  # Secret with 'token' key containing PAT
```

## ResourceSet Deep Dive

### Purpose

ResourceSet is a **templating engine** that generates Kubernetes resources based on inputs. Think of it like Helm, but:
- Inputs come from ResourceSetInputProvider (dynamic) or inline (static)
- Uses Go templating with `<< >>` delimiters (not `{{ }}` to avoid conflicts)
- Manages the lifecycle of generated resources (create, update, delete)

### Template Syntax

ResourceSet uses Go templates with custom delimiters to avoid conflicts with Kustomize/Helm:

| Standard Go | ResourceSet |
|-------------|-------------|
| `{{ .foo }}` | `<< .foo >>` |
| `{{- .foo -}}` | `<<- .foo ->>` |

Available template functions (subset):
- `quote` — Wrap in quotes
- `replace` — String replacement
- `lower`, `upper` — Case conversion
- `default` — Default value if empty
- Standard Go template functions

### How It Works

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        ResourceSet Controller                           │
│                                                                         │
│  1. Read spec from CR                                                   │
│     ┌─────────────────────────────────────────────────────────────┐     │
│     │ spec:                                                       │     │
│     │   inputsFrom:                                               │     │
│     │     - name: test-branches  # ResourceSetInputProvider       │     │
│     │   resources:               # Templates to render            │     │
│     │     - apiVersion: source.toolkit.fluxcd.io/v1               │     │
│     │       kind: GitRepository                                   │     │
│     │       metadata:                                             │     │
│     │         name: test-<< inputs.id >>                          │     │
│     │       spec:                                                 │     │
│     │         ref:                                                │     │
│     │           branch: << inputs.branch >>                       │     │
│     └─────────────────────────────────────────────────────────────┘     │
│                              │                                          │
│  2. Fetch inputs from        │                                          │
│     ResourceSetInputProvider │                                          │
│                              ▼                                          │
│     ┌─────────────────────────────────────────────┐                     │
│     │ Inputs from test-branches:                  │                     │
│     │ - { branch: "test/hello", id: "371459076" } │                     │
│     │ - { branch: "test/media", id: "892734561" } │                     │
│     └─────────────────────────────────────────────┘                     │
│                              │                                          │
│  3. For each input:          │                                          │
│     Render templates         │                                          │
│                              ▼                                          │
│     ┌─────────────────────────────────────────────┐                     │
│     │ Rendered for input[0] (test/hello):         │                     │
│     │                                             │                     │
│     │ apiVersion: source.toolkit.fluxcd.io/v1     │                     │
│     │ kind: GitRepository                         │                     │
│     │ metadata:                                   │                     │
│     │   name: test-371459076                      │                     │
│     │ spec:                                       │                     │
│     │   ref:                                      │                     │
│     │     branch: test/hello                      │                     │
│     └─────────────────────────────────────────────┘                     │
│                              │                                          │
│  4. Apply rendered resources │ kubectl apply equivalent                 │
│                              │                                          │
│  5. Update inventory         │ Track what was created                   │
│     ┌─────────────────────────────────────────────┐                     │
│     │ status.inventory.entries:                   │                     │
│     │ - id: flux-system_test-371459076_..._GitRep │                     │
│     │ - id: flux-system_test-371459076_..._Kustom │                     │
│     │ - id: flux-system_test-892734561_..._GitRep │                     │
│     │ - id: flux-system_test-892734561_..._Kustom │                     │
│     └─────────────────────────────────────────────┘                     │
│                              │                                          │
│  6. Prune orphaned resources │ If input removed, delete its resources   │
└─────────────────────────────────────────────────────────────────────────┘
```

### Inventory Management

The ResourceSet maintains an **inventory** of all resources it created. This enables:

1. **Ownership tracking** — Know which resources belong to which ResourceSet
2. **Pruning** — When an input is removed, delete its resources
3. **Drift detection** — Detect if someone manually modified a managed resource

```yaml
status:
  inventory:
    entries:
      - id: flux-system_test-371459076_source.toolkit.fluxcd.io_GitRepository
        v: v1
      - id: flux-system_test-371459076_kustomize.toolkit.fluxcd.io_Kustomization
        v: v1
```

### Pruning Behavior

When an input disappears (e.g., branch deleted):

1. ResourceSetInputProvider updates its `exportedInputs` (branch gone)
2. ResourceSet reconciles, sees fewer inputs than before
3. ResourceSet compares current inputs to inventory
4. Resources in inventory but not in inputs = **orphaned**
5. Orphaned resources are **deleted**

This is how cleanup happens automatically when you delete a test branch.

## Reconciliation Flow (Complete Picture)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            Time ──────────────────────────────────────▶     │
│                                                                             │
│  ResourceSetInputProvider                                                   │
│  ├─ T+0:00  Poll GitHub API, find test/hello                                │
│  ├─ T+0:01  Update status.exportedInputs                                    │
│  │          Trigger ResourceSet reconciliation                              │
│  │                                                                          │
│  ResourceSet                                                                │
│  ├─ T+0:02  Read inputs from InputProvider                                  │
│  ├─ T+0:02  Render templates for test/hello                                 │
│  ├─ T+0:03  Create GitRepository/test-371459076                             │
│  ├─ T+0:03  Create Kustomization/test-371459076                             │
│  ├─ T+0:04  Update inventory                                                │
│  │                                                                          │
│  GitRepository (Flux)                                                       │
│  ├─ T+0:05  Clone https://github.com/.../homelab branch:test/hello          │
│  ├─ T+0:10  Store artifact (tarball of repo at that commit)                 │
│  │                                                                          │
│  Kustomization (Flux)                                                       │
│  ├─ T+0:11  Fetch artifact from GitRepository                               │
│  ├─ T+0:12  Run kustomize build on ./manifests/apps                         │
│  ├─ T+0:13  Apply rendered manifests to test namespace                      │
│  ├─ T+0:14  Update inventory (what Kustomization created)                   │
│  │                                                                          │
│  ... Time passes, branch deleted ...                                        │
│                                                                             │
│  ResourceSetInputProvider                                                   │
│  ├─ T+5:00  Poll GitHub API, test/hello gone                                │
│  ├─ T+5:01  Update status.exportedInputs (empty)                            │
│  │          Trigger ResourceSet reconciliation                              │
│  │                                                                          │
│  ResourceSet                                                                │
│  ├─ T+5:02  Read inputs (now empty)                                         │
│  ├─ T+5:02  Compare to inventory: test-371459076 is orphaned                │
│  ├─ T+5:03  Delete Kustomization/test-371459076                             │
│  │                                                                          │
│  Kustomization (Flux) — deletion triggers finalizer                         │
│  ├─ T+5:04  Prune all resources it created in test namespace                │
│  ├─ T+5:05  Finalizer completes, Kustomization deleted                      │
│  │                                                                          │
│  ResourceSet                                                                │
│  ├─ T+5:06  Delete GitRepository/test-371459076                             │
│  ├─ T+5:07  Update inventory (now empty)                                    │
└─────────────────────────────────────────────────────────────────────────────┘
```

## API Reference

### ResourceSetInputProvider Spec

```yaml
apiVersion: fluxcd.controlplane.io/v1
kind: ResourceSetInputProvider
metadata:
  name: example
  namespace: flux-system
  annotations:
    # How often to poll the external source
    fluxcd.controlplane.io/reconcileEvery: "5m"
spec:
  # Type of input source
  type: GitHubBranch  # or GitHubPullRequest, GitLabBranch, etc.
  
  # Repository URL
  url: https://github.com/owner/repo
  
  # Optional: Authentication for private repos or rate limits
  secretRef:
    name: github-token
  
  # Filter which items to include
  filter:
    # Regex pattern for branch names
    includeBranch: "^test/.*"
    # Or for PRs: includeLabel, excludeLabel, includeDraft
  
  # Default values merged into each input
  defaultValues:
    environment: "preview"
    region: "us-east-1"
```

### ResourceSetInputProvider Status

```yaml
status:
  conditions:
    - type: Ready
      status: "True"
      reason: ReconciliationSucceeded
      message: "Reconciliation finished in 233ms"
  
  # Inputs exported to ResourceSets
  exportedInputs:
    - branch: "test/hello"
      sha: "def456abc123..."
      id: "371459076"
    - branch: "test/media"
      sha: "ghi789def456..."
      id: "892734561"
  
  # Hash of current inputs (for change detection)
  lastExportedRevision: "sha256:a64a25..."
```

### ResourceSet Spec

```yaml
apiVersion: fluxcd.controlplane.io/v1
kind: ResourceSet
metadata:
  name: example
  namespace: flux-system
spec:
  # Where to get inputs
  inputsFrom:
    - apiVersion: fluxcd.controlplane.io/v1
      kind: ResourceSetInputProvider
      name: test-branches
  
  # Optional: Static inputs (useful for testing)
  inputs:
    - name: "static-env"
      values:
        key: "value"
  
  # Optional: ServiceAccount for created resources
  serviceAccountName: flux
  
  # Templates to render for each input
  # Uses << >> delimiters, not {{ }}
  resources:
    - apiVersion: v1
      kind: Namespace
      metadata:
        name: preview-<< inputs.id >>
    
    - apiVersion: source.toolkit.fluxcd.io/v1
      kind: GitRepository
      metadata:
        name: app-<< inputs.id >>
        namespace: preview-<< inputs.id >>
      spec:
        url: https://github.com/owner/repo
        ref:
          commit: << inputs.sha >>
```

### ResourceSet Status

```yaml
status:
  conditions:
    - type: Ready
      status: "True"
      reason: ReconciliationSucceeded
  
  # Resources created by this ResourceSet
  inventory:
    entries:
      - id: flux-system_test-371459076_source.toolkit.fluxcd.io_GitRepository
        v: v1
      - id: flux-system_test-371459076_kustomize.toolkit.fluxcd.io_Kustomization
        v: v1
  
  # Reconciliation history
  history:
    - digest: "sha256:4d996f..."
      firstReconciled: "2026-07-28T21:59:29Z"
      lastReconciled: "2026-07-28T21:59:29Z"
      metadata:
        inputs: "1"
        resources: "2"
```

## Rate Limits and Performance

### GitHub API Rate Limits

| Authentication | Limit | Per Day (at 5min poll) |
|----------------|-------|------------------------|
| None | 60/hour | 288 requests |
| PAT | 5,000/hour | 288 requests |
| GitHub App | 5,000/hour | 288 requests |

**Calculation:** 60 minutes/hour ÷ 5 minutes = 12 requests/hour × 24 hours = 288/day

At 1-minute polling: 1,440/day (exactly at unauthenticated limit)

### Manual Reconciles Count

Triggering via annotation:
```bash
kubectl annotate resourcesetinputprovider test-branches \
  reconcile.fluxcd.io/requestedAt=$(date +%s) --overwrite
```

**This triggers an API call** and counts against rate limits.

### Optimizing for Performance

1. **Use longer poll intervals** — 5 minutes is good for most cases
2. **Use PAT for active development** — Avoid rate limit issues
3. **Webhook for instant feedback** — Configure GitHub webhook to trigger reconciliation (eliminates polling)

## Comparison with Alternatives

### vs Argo CD ApplicationSets

| Feature | Flux ResourceSet | Argo ApplicationSets |
|---------|------------------|---------------------|
| Git provider support | GitHub, GitLab, Azure, Gitea | GitHub, GitLab, Bitbucket |
| Template syntax | Go templates (`<< >>`) | Go templates (`{{ }}`) |
| PR generators | Yes | Yes |
| Branch generators | Yes | Yes |
| Cluster generators | No | Yes |
| Matrix generators | No | Yes |
| Merge generators | No | Yes |

Argo CD ApplicationSets has more generator types, but ResourceSet integrates natively with the Flux ecosystem.

### vs Kustomize Overlays

| Approach | Use Case |
|----------|----------|
| Kustomize overlays | Static environments (dev, staging, prod) |
| ResourceSet | Dynamic environments (per-branch, per-PR) |

ResourceSet is for when you don't know environments ahead of time.

## Further Reading

- [Flux Operator Documentation](https://fluxoperator.dev/docs/)
- [ResourceSet CRD Reference](https://fluxoperator.dev/docs/crd/resourceset/)
- [ResourceSetInputProvider CRD Reference](https://fluxoperator.dev/docs/crd/resourcesetinputprovider/)
- [Feature Branches Guide](https://fluxoperator.dev/docs/resourcesets/feature-branches/)

---

# Test Branch Workflow

This document explains how the Flux Operator enables automatic deployment of `test/*` branches for testing workloads before merging to main.

## Overview

The test branch workflow allows you to:
1. Create a `test/*` branch with new manifests
2. Push to GitHub → Flux automatically deploys to the `test` namespace
3. Validate your changes in isolation
4. Merge to main when ready
5. Delete the branch → Flux automatically cleans up test resources

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              GitHub                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                      │
│  │    main     │  │ test/hello  │  │ test/media  │                      │
│  └─────────────┘  └─────────────┘  └─────────────┘                      │
└───────────────────────────┬─────────────────────────────────────────────┘
                            │ GitHub API (list branches)
                            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         Kubernetes Cluster                              │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                      flux-system namespace                       │   │
│  │                                                                  │   │
│  │  ┌─────────────────────┐    ┌─────────────────────────────────┐  │   │
│  │  │ ResourceSetInput    │───▶│ ResourceSet                     │  │   │
│  │  │ Provider            │    │ (test-deployments)              │  │   │
│  │  │ (test-branches)     │    │                                 │  │   │
│  │  │                     │    │ For each test/* branch:         │  │   │
│  │  │ - Polls GitHub API  │    │ - Creates GitRepository         │  │   │
│  │  │ - Exports branches  │    │ - Creates Kustomization         │  │   │
│  │  │   matching test/*   │    │   (targets test namespace)      │  │   │
│  │  └─────────────────────┘    └─────────────────────────────────┘  │   │
│  │                                                                  │   │
│  │  ┌─────────────────────┐    ┌─────────────────────────────────┐  │   │
│  │  │ GitRepository       │    │ Kustomization                   │  │   │
│  │  │ (test-371459076)    │───▶│ (test-371459076)                │  │   │
│  │  │                     │    │                                 │  │   │
│  │  │ branch: test/hello  │    │ path: ./manifests/apps          │  │   │
│  │  │                     │    │ targetNamespace: test           │  │   │
│  │  └─────────────────────┘    └─────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                           │                             │
│                                           │ applies manifests           │
│                                           ▼                             │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                        test namespace                            │   │
│  │                                                                  │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐   │   │
│  │  │ ConfigMap       │  │ Deployment      │  │ Service         │   │   │
│  │  │ (hello-world)   │  │ (from branch)   │  │ (from branch)   │   │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘   │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

## Components

### 1. ResourceSetInputProvider (`test-branches`)

**Location:** `manifests/infrastructure/configs/test-branches/input-provider.yaml`

Scans GitHub for branches matching `test/*` pattern. It:
- Polls the GitHub API at a configured interval (default: 5 minutes)
- Exports branch metadata (name, SHA, ID) for the ResourceSet to consume
- Triggers ResourceSet reconciliation when branches change

```yaml
apiVersion: fluxcd.controlplane.io/v1
kind: ResourceSetInputProvider
metadata:
  name: test-branches
  namespace: flux-system
spec:
  type: GitHubBranch
  url: https://github.com/lbfrias/homelab
  filter:
    includeBranch: "^test/.*"
```

### 2. ResourceSet (`test-deployments`)

**Location:** `manifests/infrastructure/configs/test-branches/resourceset.yaml`

Templates Flux resources for each detected branch. For each `test/*` branch, it creates:
- A **GitRepository** pointing to that branch
- A **Kustomization** that deploys `./manifests/apps` to the `test` namespace

The template uses Go templating with inputs from the ResourceSetInputProvider:
- `<< inputs.branch >>` - Branch name (e.g., `test/hello`)
- `<< inputs.sha >>` - Latest commit SHA
- `<< inputs.id >>` - Unique numeric ID (used in resource names)

### 3. Test Namespace

**Location:** `manifests/infrastructure/configs/test-branches/namespace.yaml`

An isolated namespace where all test branch deployments land. Resources here don't affect production workloads.

## Usage

### Creating a Test Branch

```bash
# Create and switch to test branch
git checkout -b test/my-feature

# Add your manifests to manifests/apps/my-feature/
mkdir -p manifests/apps/my-feature
cat > manifests/apps/my-feature/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
...
EOF

# Update manifests/apps/kustomization.yaml to include your app
# (This is needed because the ResourceSet deploys from ./manifests/apps)

# Commit and push
git add -A
git commit -m "test: add my-feature deployment"
git push origin test/my-feature
```

### Triggering Immediate Deployment

By default, the ResourceSetInputProvider polls every 5 minutes. For instant feedback:

```bash
# On the jumphost (jessica)
kubectl annotate resourcesetinputprovider test-branches -n flux-system \
  reconcile.fluxcd.io/requestedAt=$(date +%s) --overwrite
```

### Checking Deployment Status

```bash
# See all test branch Kustomizations
flux get ks -A | grep test-

# Check specific test branch
kubectl get kustomization test-<ID> -n flux-system

# View resources in test namespace
kubectl get all -n test
```

### Cleaning Up

When you're done testing:

```bash
# Delete the remote branch
git push origin --delete test/my-feature

# Trigger cleanup (or wait for next poll)
kubectl annotate resourcesetinputprovider test-branches -n flux-system \
  reconcile.fluxcd.io/requestedAt=$(date +%s) --overwrite

# Verify cleanup
kubectl get all -n test
```

The ResourceSet will automatically delete the GitRepository and Kustomization, and the Kustomization will prune all resources it created in the `test` namespace.

## How It Works (Step by Step)

### Deployment Flow

1. **You push `test/my-feature`** to GitHub
2. **ResourceSetInputProvider** polls GitHub API, discovers new branch
3. **ResourceSet** receives the branch metadata as an input
4. **ResourceSet** templates and creates:
   - `GitRepository/test-<ID>` pointing to `test/my-feature` branch
   - `Kustomization/test-<ID>` deploying `./manifests/apps` to `test` namespace
5. **Kustomization** fetches manifests from the branch and applies them
6. **Your resources** appear in the `test` namespace

### Cleanup Flow

1. **You delete `test/my-feature`** branch on GitHub
2. **ResourceSetInputProvider** polls GitHub API, branch is gone
3. **ResourceSet** removes the branch from its inputs
4. **ResourceSet** deletes the GitRepository and Kustomization it created
5. **Kustomization** (with `prune: true`) deletes all resources it managed
6. **Test namespace** is clean

## Limitations

### Only Deploys `manifests/apps/`

The ResourceSet is configured to deploy from `./manifests/apps` only, not the entire `manifests/` directory. This prevents test branches from:
- Re-deploying Flux itself (`flux-system/`)
- Modifying infrastructure controllers
- Conflicting with production resources

**To test infrastructure changes**, you'll need a different approach (e.g., a separate test cluster).

### GitHub API Rate Limits

The ResourceSetInputProvider uses the GitHub REST API (not git protocol):

| Auth | Rate Limit | Requests/Day |
|------|------------|--------------|
| None (public repo) | 60/hour | 1,440/day |
| With PAT | 5,000/hour | 120,000/day |

At 5-minute polling: ~288 requests/day (well under unauthenticated limit)

**Note:** Manual reconcile triggers also count against the rate limit.

### Namespace Isolation

All test deployments go to the `test` namespace. This means:
- Resources with the same name from different branches will conflict
- Cluster-scoped resources (CRDs, ClusterRoles) cannot be tested this way
- Cross-namespace references won't work as expected

### No Private Repo Support Without PAT

For private repositories, you must create a GitHub PAT secret:

```bash
kubectl create secret generic github-token \
  -n flux-system \
  --from-literal=token=ghp_your_token_here
```

Then add `secretRef` to the ResourceSetInputProvider.

## Troubleshooting

### Branch Not Detected

1. Check ResourceSetInputProvider status:
   ```bash
   kubectl get resourcesetinputprovider test-branches -n flux-system
   ```

2. Force a reconcile:
   ```bash
   kubectl annotate resourcesetinputprovider test-branches -n flux-system \
     reconcile.fluxcd.io/requestedAt=$(date +%s) --overwrite
   ```

3. Verify branch name matches pattern (`test/*`)

### Deployment Failing

1. Check the Kustomization status:
   ```bash
   kubectl get ks -n flux-system | grep test-
   kubectl describe ks test-<ID> -n flux-system
   ```

2. Check the GitRepository:
   ```bash
   kubectl get gitrepository -n flux-system | grep test-
   ```

3. Ensure `manifests/apps/kustomization.yaml` includes your app

### Resources Not Cleaning Up

1. Force ResourceSetInputProvider reconcile (branch deletion detection)
2. Check if ResourceSet still shows the branch:
   ```bash
   kubectl get resourceset test-deployments -n flux-system -o yaml | grep -A20 inventory
   ```

3. Manually delete if stuck:
   ```bash
   kubectl delete ks test-<ID> -n flux-system
   kubectl delete gitrepository test-<ID> -n flux-system
   ```

## File Reference

```
manifests/
├── infrastructure/
│   ├── controllers/
│   │   └── flux-operator/
│   │       ├── release.yaml          # HelmRelease for Flux Operator
│   │       └── kustomization.yaml
│   └── configs/
│       └── test-branches/
│           ├── namespace.yaml        # test namespace
│           ├── input-provider.yaml   # ResourceSetInputProvider
│           ├── resourceset.yaml      # ResourceSet template
│           ├── kustomization.yaml
│           └── README.md
└── apps/
    └── kustomization.yaml            # Must include test apps
```
