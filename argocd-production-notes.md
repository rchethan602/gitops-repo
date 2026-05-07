# ArgoCD Production Reference Notes

> Staff-level reference covering architecture, core concepts, production patterns, debugging, and design trade-offs.

---

## Table of Contents

1. [How ArgoCD Works](#how-argocd-works)
2. [Core Components](#core-components)
3. [Sync Policy](#sync-policy)
4. [Sync Options](#sync-options)
5. [Prune](#prune)
6. [Self-Heal](#self-heal)
7. [Health vs Sync Status](#health-vs-sync-status)
8. [AppProjects](#appprojects)
9. [RBAC](#rbac)
10. [Repo Design](#repo-design)
11. [Kustomize with ArgoCD](#kustomize-with-argocd)
12. [Helm with ArgoCD](#helm-with-argocd)
13. [App of Apps Pattern](#app-of-apps-pattern)
14. [ApplicationSet](#applicationset)
15. [Secrets Handling](#secrets-handling)
16. [Debugging Framework](#debugging-framework)
17. [Production Setup on EKS](#production-setup-on-eks)
18. [Scaling ArgoCD](#scaling-argocd)
19. [Single vs Multi ArgoCD](#single-vs-multi-argocd)
20. [Design Trade-offs](#design-trade-offs)
21. [CI Integration](#ci-integration)
22. [Promotion Strategy](#promotion-strategy)
23. [Quick Reference Commands](#quick-reference-commands)

---

## How ArgoCD Works

ArgoCD is a GitOps controller. It continuously compares desired state (Git) against live state (cluster) and reconciles the difference.

```
Git Repository
      │
      │  ArgoCD polls (default 3min) or receives webhook
      ▼
argocd-repo-server
      │  clones repo, runs kustomize build / helm template
      ▼
Desired Manifests
      │
      │  compared against
      ▼
argocd-application-controller
      │  watches live cluster state via Kubernetes informers
      ▼
Diff detected → marks OutOfSync → syncs (if automated) or waits
```

### Key Behaviors

- ArgoCD does NOT poll continuously — it uses Kubernetes informers for live state (event-driven, instant)
- Git polling is interval-based (configurable, default 180s)
- Webhook push makes sync near-instant (recommended for prod)
- Self-heal triggers within seconds of drift detection
- History only records new Git commits, not self-heal events

---

## Core Components

| Component | Role | Scaling |
|-----------|------|---------|
| `argocd-server` | API + UI | Multiple replicas |
| `argocd-repo-server` | Clones repos, renders manifests | Multiple replicas (bottleneck at scale) |
| `argocd-application-controller` | Reconciliation loop, drift detection | Sharding for 100+ apps |
| `argocd-dex-server` | SSO/OIDC authentication | Single (stateless) |
| `argocd-redis` | Cache layer, app state | Redis HA for prod |
| `argocd-applicationset-controller` | Generates Applications from templates | Single |
| `argocd-notifications-controller` | Alerts on app events | Single |

### Component Failure Impact

```
repo-server down    → cannot sync, cannot refresh, UI shows stale state
app-controller down → no reconciliation, no drift detection, apps keep running
argocd-server down  → no UI, no CLI, no API — but existing apps keep running
redis down          → full ArgoCD outage, nothing works
```

Running pods are unaffected when ArgoCD itself is down. GitOps keeps working; management stops.

---

## Sync Policy

Controls when and how ArgoCD applies changes.

### Manual Sync (default)

```yaml
syncPolicy: {}    # empty = manual
```

ArgoCD detects drift, marks OutOfSync, waits. Human triggers sync.

### Automated Sync

```yaml
syncPolicy:
  automated:
    prune: true       # delete resources removed from Git
    selfHeal: true    # revert manual changes to cluster
```

### Sync Windows

Restrict when syncs are allowed. Applies to both automated and manual syncs by default.

```yaml
syncPolicy:
  syncWindows:
  - kind: allow
    schedule: "0 9 * * 1-5"    # Mon-Fri 9AM UTC
    duration: 8h
    applications:
    - '*'
    manualSync: true            # allow manual sync outside window for emergencies
```

### Per-Environment Recommendation

| Environment | Sync Policy | Prune | SelfHeal | SyncWindow |
|-------------|-------------|-------|----------|------------|
| dev | automated | true | true | none |
| staging | manual | false | false | none |
| prod | manual | false | false | weekdays only |

---

## Sync Options

Applied per Application. Control how `kubectl apply` behaves.

```yaml
syncPolicy:
  syncOptions:
  - CreateNamespace=true        # create namespace if missing
  - ServerSideApply=true        # use SSA instead of client-side apply (handles immutable fields better)
  - Replace=true                # use replace instead of apply (deletes and recreates)
  - ApplyOutOfSyncOnly=true     # only apply resources that are OutOfSync
  - PruneLast=true              # prune after all other resources are healthy
  - PrunePropagationPolicy=foreground  # wait for dependent resources to be deleted
  - RespectIgnoreDifferences=true      # apply ignoreDifferences rules during sync too
```

### When to use ServerSideApply vs Replace

```
ServerSideApply  → preferred default, handles most immutable field issues
Replace          → last resort, causes brief downtime (delete + recreate)
```

### Immutable Field Error

```
error: field is immutable: spec.selector
```

Cause: trying to change a selector on existing Deployment.
Fix: delete Deployment and let ArgoCD recreate it, or use `ServerSideApply=true`.

---

## Prune

Controls whether ArgoCD deletes resources that exist in cluster but not in Git.

```yaml
automated:
  prune: true    # deletes resources removed from Git
  prune: false   # leaves orphaned resources (safer)
```

### Prune Risks

- `prune: true` + accidentally deleting a file from Git = resource deleted from cluster
- PVCs, Secrets, CRDs are especially dangerous to prune automatically
- In prod: set `prune: false`, handle deletions explicitly

### Orphan Resources

Resources that exist in cluster but ArgoCD doesn't manage them:

```yaml
spec:
  ignoreDifferences: []
  orphanedResources:
    warn: true      # shows warning in UI but doesn't fail
    ignore:
    - kind: ConfigMap
      name: kube-root-ca.crt
```

---

## Self-Heal

Reverts manual changes to the cluster back to Git state.

```yaml
automated:
  selfHeal: true
```

### How it works

1. Engineer runs `kubectl scale deployment nginx --replicas=5`
2. Kubernetes informer fires event to application-controller immediately
3. Controller compares live state (5 replicas) vs Git state (2 replicas)
4. Drift detected → controller reapplies Git manifest → replicas reverted to 2
5. Happens within seconds

### Self-heal vs History

Self-heal does NOT create a new ArgoCD history entry. History only records new Git commits synced. Self-heal events are visible only in:
- `kubectl logs -n argocd statefulset/argocd-application-controller`
- Kubernetes audit logs
- ArgoCD notifications (if configured)

### When NOT to use SelfHeal

- Stateful applications where manual intervention is intentional
- During active incidents where operators need to hotfix directly
- Prod environments where drift = possible emergency fix

---

## Health vs Sync Status

Two independent status dimensions. Common mistake: treating them as the same.

| Sync Status | Meaning |
|-------------|---------|
| Synced | Git state matches live state |
| OutOfSync | Difference detected between Git and live |
| Unknown | Cannot compare — repo unreachable or render failed |

| Health Status | Meaning |
|---------------|---------|
| Healthy | All resources are ready and running |
| Progressing | Resources are being updated (rolling deploy) |
| Degraded | Resources failed (ImagePullBackOff, CrashLoop) |
| Missing | Resource expected but not found in cluster |
| Suspended | Paused intentionally |
| Unknown | Cannot determine health |

### Critical Combinations

```
Synced + Healthy    → everything is fine
Synced + Degraded   → Git applied but pods are failing (runtime issue)
OutOfSync + Healthy → drift exists but app is still running
Unknown + Healthy   → repo unreachable, cannot check — pods still running
```

---

## AppProjects

Namespace-level isolation for ArgoCD Applications. Your blast radius limiter.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: prod-project
  namespace: argocd
spec:
  description: Production environment

  # Which repos can deploy to this project
  sourceRepos:
  - https://github.com/your-org/gitops-repo
  # NOT '*' — only your org's repos

  # Which namespaces and clusters are allowed destinations
  destinations:
  - namespace: prod
    server: https://kubernetes.default.svc

  # Cluster-scoped resources allowed (whitelist)
  clusterResourceWhitelist:
  - group: ''
    kind: Namespace

  # Namespace-scoped resources allowed (whitelist)
  # If omitted, all namespace resources are allowed
  namespaceResourceWhitelist:
  - group: 'apps'
    kind: Deployment
  - group: ''
    kind: Service
  - group: ''
    kind: ConfigMap

  # Sync windows per project
  syncWindows:
  - kind: allow
    schedule: "0 9 * * 1-5"
    duration: 8h
    applications: ['*']
    manualSync: true

  # Project-level roles (alternative to global RBAC)
  roles:
  - name: prod-deployer
    policies:
    - p, proj:prod-project:prod-deployer, applications, sync, prod-project/*, allow
    groups:
    - ops-team
```

### Why Projects matter beyond RBAC

A misconfigured Application with `prune: true` targeting the wrong namespace is stopped by Project destination restrictions. Without Projects, a typo in `namespace:` can prune resources in the wrong environment. Projects are your safety net.

---

## RBAC

Two-layer system: global RBAC in `argocd-rbac-cm` + Project-level roles.

### Global RBAC

```yaml
# argocd-rbac-cm
data:
  policy.default: ""          # deny everything by default — critical for prod
  policy.csv: |
    # Built-in admin role
    p, role:admin, applications, *, */*, allow
    p, role:admin, clusters, get, *, allow
    p, role:admin, repositories, *, *, allow

    # Dev team — dev project only
    p, role:dev-team, applications, get,    dev-project/*, allow
    p, role:dev-team, applications, sync,   dev-project/*, allow
    p, role:dev-team, applications, create, dev-project/*, allow
    p, role:dev-team, repositories, get,    *, allow

    # Ops team — staging and prod
    p, role:ops-team, applications, get,    staging-project/*, allow
    p, role:ops-team, applications, sync,   staging-project/*, allow
    p, role:ops-team, applications, get,    prod-project/*, allow
    p, role:ops-team, applications, sync,   prod-project/*, allow

    # Auditors — read only everywhere
    p, role:auditor, applications, get,     */*, allow
    p, role:auditor, repositories, get,     *, allow

    # Group assignments
    g, dev-user,     role:dev-team
    g, ops-user,     role:ops-team
    g, auditor-user, role:auditor
    g, chethan,      role:admin
```

### RBAC Actions

```
get     → read/view
sync    → trigger sync
create  → create new Application
update  → modify existing Application
delete  → delete Application
action  → trigger resource actions (restart, etc.)
override → override sync policy
```

### Critical: policy.default

```yaml
policy.default: role:readonly   # DANGEROUS — every user sees everything
policy.default: ""              # CORRECT — deny by default, grant explicitly
```

### User Accounts

```yaml
# argocd-cm
accounts.username: login        # creates login-capable account
accounts.username: login,apiKey # creates account with API key capability
```

### Disable Admin

```yaml
# argocd-cm
admin.enabled: "false"
```

Never use shared admin in prod. Create named accounts. Use SSO (Dex + OIDC) for teams.

---

## Repo Design

### The Core Decision: Branch-per-env vs Path-per-env

**Branch-per-env**
```
master  → dev
staging → staging branch
release → prod
```

Problems: merge conflicts, drift between branches, cherry-picking replaces promoting, hard to audit what's in prod.

**Path-per-env (recommended)**
```
master (single branch)
├── apps/nginx/overlays/dev/
├── apps/nginx/overlays/staging/
└── apps/nginx/overlays/prod/
```

Benefits: single source of truth, clean git log, CODEOWNERS works cleanly per path.

### Mono-repo vs Poly-repo

| Pattern | Use when |
|---------|----------|
| Mono-repo | Small team, few services, simple governance |
| Poly-repo (app code + gitops config separate) | Multiple teams, compliance requirements, independent deploy cadence |

Production standard: **separate GitOps config repo from app code repo**. CI token gets write access to config repo only, not source code.

### CODEOWNERS for promotion control

```
# .github/CODEOWNERS
apps/nginx/base/              @senior-engineers    # base = touches all envs
apps/nginx/overlays/prod/     @senior-engineers    # prod needs senior approval
apps/nginx/overlays/staging/  @mid-engineers
apps/nginx/overlays/dev/      @any-engineer
```

---

## Kustomize with ArgoCD

### Structure

```
apps/
└── nginx/
    ├── base/
    │   ├── deployment.yaml    # no namespace field in base
    │   ├── service.yaml
    │   └── kustomization.yaml
    └── overlays/
        ├── dev/
        │   ├── kustomization.yaml
        │   └── replica-patch.yaml
        ├── staging/
        │   ├── kustomization.yaml
        │   └── replica-patch.yaml
        └── prod/
            ├── kustomization.yaml
            └── replica-patch.yaml
```

### Overlay kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: prod               # namespace set in overlay, not base
resources:
- ../../base
images:
- name: public.ecr.aws/nginx/nginx    # MUST match exact image name in base
  newTag: "1.26.0"
patches:
- path: replica-patch.yaml
```

### Common Mistakes

**1. commonLabels changes selector — immutable field error**
```yaml
# DON'T use commonLabels on running apps
commonLabels:
  env: dev    # this modifies spec.selector — Kubernetes rejects it
```

**2. Image name mismatch — silent skip**
```yaml
images:
- name: nginx                           # won't match
  newTag: "1.26.0"
# base has: image: public.ecr.aws/nginx/nginx  ← different name
# kustomize silently skips override — no error, wrong image
```

**3. File referenced but not pushed**
```
Error: evalsymlink failure: no such file or directory
```
Always run `kustomize build overlays/dev` locally before pushing.

### Validate Before Pushing (mandatory habit)

```bash
kustomize build apps/nginx/overlays/dev | grep -E "kind:|image:|namespace:"
```

---

## Helm with ArgoCD

### Multiple Values Files — The Core Pattern

```yaml
# ArgoCD Application
spec:
  source:
    path: charts/myapp
    helm:
      releaseName: myapp
      valueFiles:
      - values.yaml           # chart defaults
      - prod-values.yaml      # env overrides
      values: |               # inline overrides (highest priority)
        global:
          environment: prod
          region: eu-central-1
```

### Values Priority (last wins)

```
chart/values.yaml → values.yaml → prod-values.yaml → inline values → --helm-set
```

### Passing Additional Values at Deploy Time

**Option 1: Inline values in Application spec** (persisted in Git via Application yaml)
```yaml
helm:
  values: |
    feature_flag: "true"
    image:
      tag: "1.27.0"
```

**Option 2: CLI at sync time** (emergency only, not persisted)
```bash
argocd app set prod-myapp --helm-set payments.image.tag=1.25.1-hotfix
argocd app sync prod-myapp
# Always clean up after: remove --helm-set and update values file in Git
```

**Option 3: helm parameters in spec** (avoid — creates config outside values files)
```yaml
helm:
  parameters:
  - name: image.tag
    value: "1.27.0"
```

### Umbrella Chart Pattern (one chart, multiple services)

When all services deploy together in one release cycle:

```yaml
# values.yaml
nginx:
  enabled: true
  image:
    repository: public.ecr.aws/nginx/nginx
    tag: "1.26.0"
  replicaCount: 1

payments:
  enabled: true
  image:
    repository: public.ecr.aws/payments/app
    tag: "2.1.0"
  replicaCount: 1

# Disable a service per env without deleting templates
# dev-values.yaml
orders:
  enabled: false
```

```yaml
# templates/nginx-deployment.yaml
{{- if .Values.nginx.enabled }}
apiVersion: apps/v1
kind: Deployment
...
{{- end }}
```

### Umbrella Chart Trade-offs

| Pro | Con |
|-----|-----|
| One `helm upgrade` deploys everything | All services restart on every upgrade |
| Consistent release versioning | Cannot roll back one service independently |
| Simple operations | One broken template blocks entire release |

Use umbrella chart when: all services always deploy together, shared release cycle.
Use separate charts when: services deploy independently, different teams own different services.

### Validate Before Pushing

```bash
helm lint charts/myapp -f values.yaml -f prod-values.yaml
helm template myapp charts/myapp -f values.yaml -f prod-values.yaml \
  --namespace prod | grep -E "image:|replicas:|memory:"
```

---

## App of Apps Pattern

Manage all ArgoCD Applications from Git. Bootstrap once, everything else is GitOps.

```yaml
# Root Application — apply this once manually
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io  # cascade delete — be careful
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/gitops-repo
    targetRevision: master
    path: argocd-apps               # directory containing all Application yamls
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: false      # IMPORTANT: don't auto-delete child apps
      selfHeal: true
```

### Blast Radius of Root App

Root app manages everything. If it prunes incorrectly:
- Child Application deleted → cascade finalizer fires → all resources in that namespace deleted
- Always use `prune: false` on root app
- Deletions of Applications must be explicit and reviewed

### Repo structure

```
gitops-repo/
├── argocd-apps/
│   ├── root-app.yaml           # bootstrap only
│   ├── projects.yaml
│   ├── dev-nginx.yaml
│   ├── staging-nginx.yaml
│   └── prod-nginx.yaml
└── apps/
    └── nginx/
```

---

## ApplicationSet

Dynamically generates Applications. Better than App of Apps at scale.

### Matrix Generator — environments × services

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: all-services
  namespace: argocd
spec:
  goTemplate: true
  generators:
  - matrix:
      generators:
      # Discovers service directories automatically
      - git:
          repoURL: https://github.com/your-org/gitops-repo
          revision: master
          directories:
          - path: services/*
      # Environment matrix
      - list:
          elements:
          - env: dev
            namespace: dev
            project: dev-project
            syncPolicy: automated
          - env: staging
            namespace: staging
            project: staging-project
            syncPolicy: manual
          - env: prod
            namespace: prod
            project: prod-project
            syncPolicy: manual
  template:
    metadata:
      name: '{{ .path.basename }}-{{ .env }}'
    spec:
      project: '{{ .project }}'
      source:
        repoURL: https://github.com/your-org/gitops-repo
        targetRevision: master
        path: charts/microservice
        helm:
          releaseName: '{{ .path.basename }}'
          valueFiles:
          - '../../services/{{ .path.basename }}/values.yaml'
          - '../../services/{{ .path.basename }}/{{ .env }}-values.yaml'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{ .namespace }}'
```

Adding a new service = create `services/new-service/` directory with values files. ApplicationSet auto-generates 3 Applications (dev/staging/prod). No manual Application yaml creation.

### App of Apps vs ApplicationSet

| Pattern | Use when |
|---------|----------|
| App of Apps | Simple setup, explicit control, easy to understand |
| ApplicationSet | 10+ services, dynamic generation, PR-based previews |

---

## Secrets Handling

### The Core Problem

Git is the source of truth. Secrets cannot be in Git (permanent history, anyone with repo access sees them).

### Solution: External Secrets Operator (ESO) + AWS Secrets Manager

```
AWS Secrets Manager  →  ESO  →  K8s Secret  →  Pod
      (value)           (sync)   (reference)   (consumes)

Git only has: ExternalSecret CR (a pointer, not a value)
```

### ExternalSecret

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secret
  namespace: prod
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: app-secret              # K8s secret name created
    creationPolicy: Owner         # ESO owns lifecycle — deletes K8s secret if ExternalSecret deleted
  data:
  - secretKey: DB_PASSWORD        # key in K8s secret
    remoteRef:
      key: gitops/prod/app-secret # AWS secret name
      property: DB_PASSWORD       # field in JSON secret
```

### ClusterSecretStore (shared across namespaces)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: eu-central-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets
```

### Secret Rotation Gap

```
Rotate secret in AWS
  → ESO syncs K8s secret (up to refreshInterval = 1h)
    → K8s secret updated
      → Pod still has OLD value (env vars set at startup)
        → Pod must restart to pick up new value
```

### Fix: Reloader

```yaml
# Deployment annotation
metadata:
  annotations:
    secret.reloader.stakater.com/reload: "app-secret"
# Reloader watches K8s secret → auto-rolling-restart when secret changes
```

### Env vars vs Volume mounts

```
Env vars       → set at pod startup, requires restart to update
Volume mounts  → Kubernetes updates them automatically when K8s secret changes
               → app must re-read the file (most apps don't do this by default)
```

### IRSA for ESO (never use node instance profile)

```bash
# Node instance profile = all pods on node get AWS access
# IRSA = only ESO service account gets Secrets Manager access
eksctl create iamserviceaccount \
  --name external-secrets-sa \
  --namespace external-secrets \
  --cluster your-cluster \
  --attach-policy-arn arn:aws:iam::ACCOUNT:policy/ESO-SecretsManager-Policy \
  --approve
```

### ArgoCD's visibility into secrets

ArgoCD manages `ExternalSecret` CR — the pointer. It does NOT manage the K8s secret ESO creates. If ESO fails to sync, ArgoCD shows `Healthy` because the ExternalSecret CR exists. The missing K8s secret is invisible to ArgoCD.

Monitor with:
```bash
kubectl get externalsecret -A | grep -v "True"   # shows failing ExternalSecrets
```

---

## Debugging Framework

### The Four Failure Layers

```
Layer 1: Git           → ArgoCD cannot read repo
Layer 2: Manifest      → kustomize/helm render fails
Layer 3: Kubernetes API → kubectl apply fails
Layer 4: Runtime       → Pod starts but crashes
```

### Debugging Sequence

```bash
# Step 1: Locate failure layer
argocd app get <app>              # read Conditions + resource status table

# Step 2: Git/manifest layer
argocd app diff <app>             # what changed?
kustomize build <path>            # does it render locally?
helm template <release> <chart> -f values.yaml   # does helm render?

# Step 3: Kubernetes API layer
argocd app sync <app> --dry-run   # would apply succeed?
kubectl describe <resource> -n <ns>   # what did API server reject?

# Step 4: Runtime layer
kubectl get pods -n <ns>
kubectl describe pod <pod> -n <ns>     # Events section — image pull, scheduling
kubectl logs <pod> -n <ns>
kubectl logs <pod> -n <ns> --previous  # if pod restarted

# Step 5: ArgoCD internals
kubectl logs -n argocd deployment/argocd-repo-server --tail=50
kubectl logs -n argocd statefulset/argocd-application-controller --tail=50
```

### Common Errors and Root Causes

| Error | Layer | Root Cause |
|-------|-------|-----------|
| `authentication required: Repository not found` | Git | Wrong credentials or repo URL |
| `unable to resolve 'main' to a commit SHA` | Git | Branch doesn't exist or repo is empty |
| `context deadline exceeded` | Git | Network timeout to GitHub — transient or egress blocked |
| `kustomize build failed: no such file` | Manifest | File referenced in kustomization.yaml not pushed |
| `kustomization.yaml is empty` | Manifest | File exists but has no content |
| `field is immutable: spec.selector` | K8s API | Trying to change selector on existing Deployment |
| `resource not permitted in project` | K8s API | AppProject doesn't allow this resource kind or namespace |
| `ImagePullBackOff` | Runtime | Image tag doesn't exist in registry |
| `CrashLoopBackOff` | Runtime | App starts and crashes — check logs |
| `secret not found` | Runtime | Pod references missing secret — ESO failed |
| `SecretSyncedError` | Infrastructure | ESO cannot reach AWS — wrong region, bad IRSA |

### Emergency Techniques

```bash
# Force immediate refresh (bypass poll interval and cache)
argocd app refresh <app> --hard

# Sync to specific known-good commit SHA
argocd app sync <app> --revision <sha>

# Sync only OutOfSync resources (skip already-synced)
argocd app sync <app> --sync-option ApplyOutOfSyncOnly=true

# Dry run before applying
argocd app sync <app> --dry-run

# Force ESO to re-sync secret immediately
kubectl annotate externalsecret <name> -n <ns> \
  force-sync=$(date +%s) --overwrite
```

---

## Production Setup on EKS

### Installation (Helm)

```yaml
# argocd-values.yaml
configs:
  params:
    server.insecure: true         # when TLS terminated at ALB/ingress
    application.namespaces: "dev,staging,prod"

  cm:
    admin.enabled: "false"        # disable shared admin
    accounts.yourname: login
    users.anonymous.enabled: "false"
    statusbadge.enabled: "false"
    timeout.reconciliation: 300s  # 5 min polling (use webhooks instead)
    timeout.reconciliation.jitter: 60s

    # Exclude noisy resources from tracking
    resource.exclusions: |
      - apiGroups: ['', 'discovery.k8s.io']
        kinds: [Endpoints, EndpointSlice]
      - apiGroups: ['coordination.k8s.io']
        kinds: [Lease]

  rbac:
    policy.default: ""            # deny by default
    policy.csv: |
      g, yourname, role:admin

server:
  service:
    type: ClusterIP               # never public LoadBalancer

repoServer:
  replicas: 2
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      memory: 512Mi

applicationSet:
  enabled: true

notifications:
  enabled: true    # wire to Slack/PagerDuty
```

### What NOT to do in prod

```
❌ server.service.type: LoadBalancer  → exposes GitOps control plane publicly
❌ admin.enabled: "true"             → shared credentials, no audit trail
❌ policy.default: role:readonly     → everyone sees everything
❌ prune: true on prod               → auto-deletes prod resources
❌ selfHeal: true on prod            → auto-reverts emergency fixes
❌ float Helm chart version          → CRD schema changes break upgrade
❌ commonLabels in Kustomize         → breaks immutable selectors on existing apps
❌ image: latest                     → non-deterministic, defeats GitOps
```

### Security Checklist

```
✅ admin account disabled
✅ Named accounts with individual passwords
✅ SSO/OIDC for teams (Dex + Okta/GitHub/Google)
✅ policy.default: "" (deny by default)
✅ AppProjects per environment
✅ IRSA for ESO (not node instance profile)
✅ Secrets in AWS Secrets Manager (not Git)
✅ argocd-initial-admin-secret deleted after bootstrap
✅ GitHub PAT scoped to minimum permissions
✅ No direct kubectl access to prod for most engineers
✅ Kubernetes RBAC aligned with ArgoCD RBAC
```

---

## Scaling ArgoCD

### Performance Bottlenecks (in order of impact)

```
1. No webhooks → polling every 3min → constant load
2. Single repo-server → serialized manifest generation
3. Single app-controller → all 100 apps in one queue
4. Single Redis → cache bottleneck
```

### Fix In Order

**1. Webhooks first (free, biggest impact)**
```
GitHub → push event → ArgoCD /api/webhook → immediate refresh
```

**2. Reduce reconciliation interval**
```yaml
timeout.reconciliation: 300s    # 5 min
```

**3. Scale repo-server**
```yaml
repoServer:
  replicas: 3
```

**4. Shard application-controller (100+ apps)**
```yaml
applicationController:
  replicas: 3
  env:
  - name: ARGOCD_CONTROLLER_SHARDING_ALGORITHM
    value: consistent-hashing
```

**5. Redis HA (500+ apps)**
```yaml
redis-ha:
  enabled: true
```

**6. Separate ArgoCD per cluster (1000+ apps)**

---

## Single vs Multi ArgoCD

### Decision Framework

```
Same security boundary (all dev)?    → single ArgoCD acceptable
Prod + non-prod in same instance?    → always separate
Different teams own different clusters? → separate per team
Cross-region clusters?               → prefer regional ArgoCD (latency + blast radius)
Regulated workloads (PCI/SOC2)?      → separate per compliance boundary
```

### Hub and Spoke (recommended at scale)

```
Management cluster
└── ArgoCD (manages only ArgoCD deployments)
      ├── deploys ArgoCD → prod-cluster
      ├── deploys ArgoCD → staging-cluster
      └── deploys ArgoCD → dev-cluster

prod-cluster-argocd     → manages prod workloads only
staging-cluster-argocd  → manages staging workloads only
dev-cluster-argocd      → manages dev workloads only
```

Management ArgoCD going down = can't update ArgoCD configs, but all workloads keep running.

---

## Design Trade-offs

### Promotion: Git-only vs ArgoCD sync command

```
Git-only promotion:
  Edit overlay file → PR → review → merge → ArgoCD detects → manual sync
  Pro: full audit trail in Git, approval in Git
  Con: slower process

ArgoCD CLI promotion:
  argocd app sync prod-nginx
  Pro: fast
  Con: no PR, no review, no Git history of the decision
```

### Kustomize vs Helm

| Scenario | Use |
|----------|-----|
| Simple env differences, plain YAML | Kustomize |
| Complex conditionals, loops, packaging | Helm |
| Third-party app (Prometheus, Cert-manager) | Helm |
| Internal apps with shared structure | Either |
| Umbrella (all services deploy together) | Helm |
| Independent service overlays | Kustomize |

### ApplicationSet vs App of Apps

| Pattern | Use when |
|---------|----------|
| App of Apps | Small setup, explicit control, easy to reason about |
| ApplicationSet | 10+ services, dynamic generation, self-service teams |

### Self-service teams — full stack

```
ApplicationSet    → auto-generates Applications from Git directories
AppProject        → restricts repos, namespaces, resource kinds per team
ResourceQuota     → limits resource consumption per namespace
LimitRange        → enforces default limits (no unlimited pods)
Kyverno/OPA       → admission control ArgoCD can't bypass
IRSA per team     → scoped AWS access per team
CODEOWNERS        → Git-level approval gates
Naming conventions → enforced by automation not humans
```

---

## CI Integration

### Correct Pattern: CI → Git → ArgoCD

```
Code push → CI builds image → pushes to ECR
→ CI updates GitOps repo (image tag in overlay)
→ git push → ArgoCD detects → syncs
```

### Wrong Pattern: CI → ArgoCD API directly

```
❌ CI calls argocd app sync directly
```

This bypasses Git entirely. If ArgoCD restarts and self-heals, it reverts to Git state (old image). GitOps breaks down.

### Update image tag in CI (use kustomize, not sed)

```bash
# ❌ Fragile
sed -i "s|newTag:.*|newTag: \"$IMAGE_TAG\"|" overlays/dev/kustomization.yaml

# ✅ Correct
cd apps/nginx/overlays/dev
kustomize edit set image public.ecr.aws/nginx/nginx=$ECR_URI:$IMAGE_TAG
```

### Trigger immediate sync after Git push

```bash
# In CI pipeline (acceptable - just refresh, not direct sync)
argocd app refresh dev-nginx --hard
# Or configure GitHub webhook for automatic trigger
```

### GitHub Actions skeleton

```yaml
- name: Update GitOps repo
  env:
    IMAGE_TAG: ${{ github.sha }}
    GITOPS_TOKEN: ${{ secrets.GITOPS_PAT }}
  run: |
    git clone https://x-token:$GITOPS_TOKEN@github.com/org/gitops-repo
    cd gitops-repo/apps/nginx/overlays/dev
    kustomize edit set image public.ecr.aws/nginx/nginx=$ECR_URI:$IMAGE_TAG
    git config user.email "ci-bot@company.com"
    git config user.name "CI Bot"
    git add kustomization.yaml
    git commit -m "ci(dev): update nginx image to $IMAGE_TAG"
    git push
```

---

## Promotion Strategy

### Git-only Promotion Flow

```
1. CI builds image → pushes to ECR
2. CI updates overlays/dev/kustomization.yaml with new tag
3. ArgoCD auto-syncs dev
4. Dev verified healthy
5. Engineer raises PR: update overlays/staging/kustomization.yaml
6. PR reviewed (CODEOWNERS) → merged
7. ArgoCD detects staging OutOfSync
8. Engineer manually syncs staging-nginx
9. Staging verified healthy
10. Engineer raises PR: update overlays/prod/kustomization.yaml
11. Senior engineer approves PR → merged
12. Engineer manually syncs prod-nginx (within sync window)
```

### Git Rollback (always, not kubectl rollout undo)

```bash
# Revert creates a new commit preserving history
git revert HEAD
git push
argocd app sync prod-nginx

# NOT this — rewrites history, blocked on protected branches
git reset --hard HEAD~1
git push --force   # ❌
```

### Emergency: Sync to specific SHA

```bash
# When GitHub is unreachable or you need immediate specific version
argocd app sync prod-nginx --revision <known-good-sha>
# Then fix Git properly afterwards
```

---

## Quick Reference Commands

### App Management

```bash
# List all apps
argocd app list

# Get app status
argocd app get <app>

# Force refresh (bypass cache and poll interval)
argocd app refresh <app> --hard

# Sync app
argocd app sync <app>

# Sync to specific revision
argocd app sync <app> --revision <sha>

# Dry run
argocd app sync <app> --dry-run

# Show diff between Git and live
argocd app diff <app>

# Show history
argocd app history <app>

# Set sync policy
argocd app set <app> --sync-policy automated --self-heal --auto-prune
argocd app set <app> --sync-policy none

# Override image
argocd app set <app> --helm-set image.tag=1.27.0
```

### Project Management

```bash
argocd proj list
argocd proj get <project>
argocd proj create <project>
```

### Account Management

```bash
argocd account list
argocd account update-password --account <name> --new-password <pass>
argocd account get-user-info
```

### Repository Management

```bash
argocd repo list
argocd repo add <url> --username <user> --password <token>
```

### Debugging

```bash
# ArgoCD component logs
kubectl logs -n argocd deployment/argocd-server --tail=50
kubectl logs -n argocd deployment/argocd-repo-server --tail=50
kubectl logs -n argocd statefulset/argocd-application-controller --tail=50

# Check live configmaps
kubectl get configmap argocd-cm -n argocd -o yaml
kubectl get configmap argocd-rbac-cm -n argocd -o yaml

# Check ESO
kubectl get externalsecret -A
kubectl describe externalsecret <name> -n <ns>
kubectl get clustersecretstore

# Kustomize local validation
kustomize build apps/nginx/overlays/dev | grep -E "kind:|image:|namespace:"

# Helm local validation
helm lint charts/myapp -f values.yaml -f prod-values.yaml
helm template myapp charts/myapp -f values.yaml -f prod-values.yaml \
  --namespace prod | grep -E "image:|replicas:|memory:"
```

---

*Reference built from hands-on EKS training covering Phases 1–11: Secure Installation, GitOps Core, Multi-Environment, Repo Design, Promotion Strategy, Secrets, Debugging, RBAC, CI Integration, Advanced Patterns, and Production Thinking.*
