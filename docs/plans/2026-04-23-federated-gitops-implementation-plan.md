# AGW Federated GitOps Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the federated GitOps reference architecture as 3 deployable repos (infra + 2 cluster repos) with ArgoCD bootstrap scripts, decomposing the enrollment-agent demo across platform/developer ownership boundaries.

**Architecture:** Hub cluster (cluster1) runs ArgoCD which manages 2 leaf clusters (cluster2, cluster3) via ApplicationSets. Helm for product installs (Istio, AGW, Kyverno, monitoring), Kustomize for authored config (backends, policies, mesh, routes). Kyverno enforces admission policies at runtime.

**Tech Stack:** ArgoCD, Kustomize, Helm, Kyverno, Enterprise Agentgateway, Istio Ambient Mesh, Prometheus/Grafana

**Base path:** `/Users/alexly-solo/Desktop/solo/solo-github/agentgateway-gitops/`

**Design spec:** `agentgateway-federated-gitops-ref-arch/docs/specs/2026-04-23-federated-gitops-implementation-design.md`

**Reference source:** `enrollment-agent/` (k8s manifests, install.sh)

---

## File Structure

### `agw-federated-infra/`

| File | Responsibility |
|------|---------------|
| `argocd/bootstrap/namespace.yaml` | ArgoCD namespace |
| `argocd/bootstrap/values.yaml` | ArgoCD Helm values |
| `argocd/projects/platform.yaml` | AppProject for infra apps |
| `argocd/projects/developers.yaml` | AppProject for cluster apps |
| `argocd/applicationsets/istio.yaml` | Istio Helm ApplicationSet (waves 1-4) |
| `argocd/applicationsets/agentgateway.yaml` | AGW Helm ApplicationSet (waves 5-6) |
| `argocd/applicationsets/enforcement.yaml` | Kyverno Helm ApplicationSet (wave 7) |
| `argocd/applicationsets/monitoring.yaml` | Prometheus/Grafana Helm ApplicationSet (wave 8) |
| `argocd/applicationsets/infra-config.yaml` | Kustomize infra config ApplicationSet (wave 10) |
| `argocd/applicationsets/cluster-apps.yaml` | Kustomize cluster apps ApplicationSet (wave 20) |
| `helm-apps/gateway-api-crds/base-values.yaml` | Gateway API CRD version pin |
| `helm-apps/istio/base-values.yaml` | Shared Istio values (hub, tag, variant) |
| `helm-apps/istio/istiod-values.yaml` | istiod-specific values |
| `helm-apps/istio/cni-values.yaml` | CNI values |
| `helm-apps/istio/ztunnel-values.yaml` | ztunnel values |
| `helm-apps/istio/overlays/leaf-1-istiod-values.yaml` | Leaf-1 cluster identity |
| `helm-apps/istio/overlays/leaf-2-istiod-values.yaml` | Leaf-2 cluster identity |
| `helm-apps/enterprise-agentgateway/crds-values.yaml` | AGW CRDs version |
| `helm-apps/enterprise-agentgateway/controller-values.yaml` | AGW controller config |
| `helm-apps/kyverno/values.yaml` | Kyverno config |
| `helm-apps/monitoring/kube-prometheus-stack-values.yaml` | Prometheus/Grafana config |
| `base/kustomization.yaml` | Root kustomization |
| `base/namespaces.yaml` | wgu-demo + wgu-demo-frontend |
| `base/agw-config/kustomization.yaml` | AGW config kustomization |
| `base/agw-config/gateway-parameters.yaml` | EnterpriseAgentgatewayParameters |
| `base/agw-config/gateway.yaml` | Gateway (agentgateway-proxy) |
| `base/agw-config/access-logging.yaml` | Access log policy |
| `base/agw-config/tracing.yaml` | Tracing policy |
| `base/backends/kustomization.yaml` | Backends kustomization |
| `base/backends/openai.yaml` | OpenAI AgentgatewayBackend |
| `base/backends/mcp-backend.yaml` | MCP AgentgatewayBackend |
| `base/global-policies/kustomization.yaml` | Global policies kustomization |
| `base/global-policies/guardrails.yaml` | PII + injection guardrails |
| `base/global-policies/rate-limit.yaml` | Token rate limiting |
| `base/mesh/kustomization.yaml` | Mesh kustomization |
| `base/mesh/deny-all.yaml` | Deny-all baseline |
| `base/mesh/waypoint.yaml` | Waypoint + telemetry |
| `base/mesh/chatbot-to-data-product.yaml` | AuthorizationPolicy |
| `base/mesh/data-product-to-graphdb.yaml` | AuthorizationPolicy |
| `base/mesh/gateway-to-financial-aid.yaml` | AuthorizationPolicy |
| `base/mesh/ingress-to-chatbot.yaml` | AuthorizationPolicy |
| `base/mesh/waypoint-to-backends.yaml` | AuthorizationPolicy |
| `base/ingress/kustomization.yaml` | Ingress kustomization |
| `base/ingress/ingress-gateway.yaml` | Ingress Gateway |
| `base/ingress/ingress-parameters.yaml` | Ingress parameters |
| `base/admission-policies/kustomization.yaml` | Admission kustomization |
| `base/admission-policies/deny-local-backends.yaml` | Kyverno: block rogue backends |
| `base/admission-policies/deny-policy-override.yaml` | Kyverno: block policy override |
| `base/admission-policies/require-guardrail-ref.yaml` | Kyverno: require guardrail |
| `base/rbac/kustomization.yaml` | RBAC kustomization |
| `base/rbac/platform-admin-role.yaml` | Platform admin ClusterRole |
| `base/rbac/developer-role.yaml` | Developer ClusterRole |
| `base/secrets/kustomization.yaml` | Secrets kustomization |
| `base/secrets/external-secrets.yaml` | ExternalSecret placeholder |
| `base/observability/kustomization.yaml` | Observability kustomization |
| `base/observability/pod-monitor.yaml` | AGW PodMonitor |
| `base/observability/grafana-dashboard-configmap.yaml` | Grafana dashboard ConfigMap |
| `overlays/leaf-1/kustomization.yaml` | Leaf-1 overlay |
| `overlays/leaf-1/patches/backend-endpoints.yaml` | Leaf-1 secret ref patch |
| `overlays/leaf-2/kustomization.yaml` | Leaf-2 overlay |
| `overlays/leaf-2/patches/backend-endpoints.yaml` | Leaf-2 secret ref patch |
| `scripts/install-argocd.sh` | Bootstrap ArgoCD on hub |
| `scripts/register-clusters.sh` | Register leaf clusters |
| `scripts/validate.sh` | End-to-end validation |
| `README.md` | Infra repo docs |

### `agw-federated-cluster-1/`

| File | Responsibility |
|------|---------------|
| `team-enrollment/kustomization.yaml` | Root kustomization |
| `team-enrollment/services/kustomization.yaml` | Services kustomization |
| `team-enrollment/services/enrollment-chatbot.yaml` | Chatbot SA + Deployment + Service |
| `team-enrollment/services/data-product-api.yaml` | Data product SA + Deployment + Service |
| `team-enrollment/services/financial-aid-mcp.yaml` | Financial aid SA + Deployment + Service |
| `team-enrollment/services/graph-db-mock.yaml` | Graph DB SA + Deployment + Service |
| `team-enrollment/services/abac-ext-authz.yaml` | ABAC ext-authz Deployment + Service |
| `team-enrollment/routes/kustomization.yaml` | Routes kustomization |
| `team-enrollment/routes/httproute-openai.yaml` | OpenAI HTTPRoute |
| `team-enrollment/routes/httproute-mcp.yaml` | MCP HTTPRoute |
| `team-enrollment/routes/ingress-routes.yaml` | Ingress HTTPRoutes |
| `team-enrollment/routes/reference-grant.yaml` | Cross-namespace ReferenceGrant |
| `team-enrollment/policies/kustomization.yaml` | Policies kustomization |
| `team-enrollment/policies/chatbot-rbac.yaml` | ClusterRole + ClusterRoleBinding |
| `argocd/appproject.yaml` | AppProject with namespace restrictions |
| `README.md` | Cluster-1 repo docs |

### `agw-federated-cluster-2/`

Same structure as cluster-1.

---

## Task 1: Create Repos and Directory Scaffolding

**Files:**
- Create: all directories and `.gitkeep` files for the 3 repos
- Create: `agw-federated-infra/.gitignore`
- Create: `agw-federated-cluster-1/.gitignore`
- Create: `agw-federated-cluster-2/.gitignore`

- [ ] **Step 1: Create parent directory and repo directories**

```bash
mkdir -p /Users/alexly-solo/Desktop/solo/solo-github/agentgateway-gitops
cd /Users/alexly-solo/Desktop/solo/solo-github/agentgateway-gitops

# Infra repo directories
mkdir -p agw-federated-infra/{argocd/{bootstrap,projects,applicationsets},helm-apps/{gateway-api-crds,istio/overlays,enterprise-agentgateway,kyverno,monitoring},base/{agw-config,backends,global-policies,mesh,ingress,admission-policies,rbac,secrets,observability},overlays/{leaf-1/patches,leaf-2/patches},scripts}

# Cluster-1 repo directories
mkdir -p agw-federated-cluster-1/{team-enrollment/{services,routes,policies},argocd}

# Cluster-2 repo directories
mkdir -p agw-federated-cluster-2/{team-enrollment/{services,routes,policies},argocd}
```

- [ ] **Step 2: Initialize git repos**

```bash
cd /Users/alexly-solo/Desktop/solo/solo-github/agentgateway-gitops

for repo in agw-federated-infra agw-federated-cluster-1 agw-federated-cluster-2; do
  cd "$repo"
  git init
  echo "*.DS_Store" > .gitignore
  git add .gitignore
  git commit -m "Initial commit"
  cd ..
done
```

- [ ] **Step 3: Create GitHub repos and set remotes**

```bash
cd /Users/alexly-solo/Desktop/solo/solo-github/agentgateway-gitops

for repo in agw-federated-infra agw-federated-cluster-1 agw-federated-cluster-2; do
  gh repo create "ably77/$repo" --public --description "AGW Federated GitOps Reference Architecture" --confirm || true
  cd "$repo"
  git remote add origin "https://github.com/ably77/$repo.git" 2>/dev/null || true
  cd ..
done
```

- [ ] **Step 4: Commit**

```bash
# Already committed in step 2
```

---

## Task 2: Infra Repo -- ArgoCD Bootstrap Configuration

**Files:**
- Create: `agw-federated-infra/argocd/bootstrap/namespace.yaml`
- Create: `agw-federated-infra/argocd/bootstrap/values.yaml`
- Create: `agw-federated-infra/argocd/projects/platform.yaml`
- Create: `agw-federated-infra/argocd/projects/developers.yaml`

- [ ] **Step 1: Create ArgoCD namespace**

Create `agw-federated-infra/argocd/bootstrap/namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: argocd
```

- [ ] **Step 2: Create ArgoCD Helm values**

Create `agw-federated-infra/argocd/bootstrap/values.yaml`:

```yaml
# ArgoCD Helm values for hub cluster
# Chart: argo/argo-cd
# Repo: https://argoproj.github.io/argo-helm
server:
  service:
    type: ClusterIP
  extraArgs:
    - --insecure  # TLS termination handled externally; for local dev only
configs:
  params:
    # Allow ApplicationSets to create apps in any namespace
    applicationsetcontroller.policy: sync
    applicationsetcontroller.enable.new.git.file.globbing: "true"
  cm:
    # Allow tracking Helm OCI charts
    helm.enabled: "true"
    # Timeout for sync operations
    timeout.reconciliation: 180s
    resource.exclusions: |
      - apiGroups:
          - cilium.io
        kinds:
          - CiliumIdentity
        clusters:
          - "*"
  repositories:
    # Solo Istio Helm charts
    solo-istio-helm:
      type: helm
      name: solo-istio-helm
      url: us-docker.pkg.dev/soloio-img/istio-helm
      enableOCI: "true"
    # Enterprise Agentgateway charts
    enterprise-agentgateway:
      type: helm
      name: enterprise-agentgateway
      url: us-docker.pkg.dev/solo-public/enterprise-agentgateway/charts
      enableOCI: "true"
    # Kyverno
    kyverno:
      type: helm
      name: kyverno
      url: https://kyverno.github.io/kyverno/
    # Prometheus community
    prometheus-community:
      type: helm
      name: prometheus-community
      url: https://prometheus-community.github.io/helm-charts
    # Infra repo
    agw-federated-infra:
      type: git
      name: agw-federated-infra
      url: https://github.com/ably77/agw-federated-infra.git
    # Cluster repos
    agw-federated-cluster-1:
      type: git
      name: agw-federated-cluster-1
      url: https://github.com/ably77/agw-federated-cluster-1.git
    agw-federated-cluster-2:
      type: git
      name: agw-federated-cluster-2
      url: https://github.com/ably77/agw-federated-cluster-2.git
applicationSet:
  enabled: true
controller:
  resources:
    requests:
      cpu: 250m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi
```

- [ ] **Step 3: Create platform AppProject**

Create `agw-federated-infra/argocd/projects/platform.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: platform
  namespace: argocd
spec:
  description: Platform team -- infra products and configuration
  sourceRepos:
    - https://github.com/ably77/agw-federated-infra.git
    - us-docker.pkg.dev/soloio-img/istio-helm/*
    - us-docker.pkg.dev/solo-public/enterprise-agentgateway/charts/*
    - https://kyverno.github.io/kyverno/
    - https://prometheus-community.github.io/helm-charts
    - https://github.com/kubernetes-sigs/gateway-api/*
  destinations:
    - namespace: '*'
      server: '*'
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
```

- [ ] **Step 4: Create developers AppProject**

Create `agw-federated-infra/argocd/projects/developers.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: developers
  namespace: argocd
spec:
  description: Developer teams -- workloads and routes
  sourceRepos:
    - https://github.com/ably77/agw-federated-cluster-1.git
    - https://github.com/ably77/agw-federated-cluster-2.git
  destinations:
    - namespace: wgu-demo
      server: '*'
    - namespace: wgu-demo-frontend
      server: '*'
    - namespace: agentgateway-system
      server: '*'
    - namespace: monitoring
      server: '*'
  clusterResourceWhitelist:
    - group: rbac.authorization.k8s.io
      kind: ClusterRole
    - group: rbac.authorization.k8s.io
      kind: ClusterRoleBinding
  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
```

- [ ] **Step 5: Commit**

```bash
cd /Users/alexly-solo/Desktop/solo/solo-github/agentgateway-gitops/agw-federated-infra
git add argocd/
git commit -m "feat: add ArgoCD bootstrap config, AppProjects"
```

---

## Task 3: Infra Repo -- ApplicationSets

**Files:**
- Create: `agw-federated-infra/argocd/applicationsets/istio.yaml`
- Create: `agw-federated-infra/argocd/applicationsets/agentgateway.yaml`
- Create: `agw-federated-infra/argocd/applicationsets/enforcement.yaml`
- Create: `agw-federated-infra/argocd/applicationsets/monitoring.yaml`
- Create: `agw-federated-infra/argocd/applicationsets/infra-config.yaml`
- Create: `agw-federated-infra/argocd/applicationsets/cluster-apps.yaml`

- [ ] **Step 1: Create Istio ApplicationSet**

Create `agw-federated-infra/argocd/applicationsets/istio.yaml`:

```yaml
# Istio Ambient Mesh components -- deployed to all leaf clusters
# Sync waves 1-4 ensure correct ordering: base -> CNI -> istiod -> ztunnel
---
# Gateway API CRDs (raw URL apply, wave 1)
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: gateway-api-crds
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            agw-role: leaf
  template:
    metadata:
      name: 'gateway-api-crds-{{name}}'
      annotations:
        argocd.argoproj.io/sync-wave: "1"
    spec:
      project: platform
      source:
        repoURL: https://github.com/kubernetes-sigs/gateway-api.git
        targetRevision: v1.5.0
        path: config/crd/experimental
        directory:
          recurse: true
      destination:
        server: '{{server}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=false
          - ServerSideApply=true
          - Replace=true
---
# Istio base (CRDs, wave 1)
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: istio-base
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            agw-role: leaf
  template:
    metadata:
      name: 'istio-base-{{name}}'
      annotations:
        argocd.argoproj.io/sync-wave: "1"
    spec:
      project: platform
      sources:
        - repoURL: us-docker.pkg.dev/soloio-img/istio-helm
          chart: base
          targetRevision: 1.29.0-solo
          helm:
            valueFiles:
              - $values/helm-apps/istio/base-values.yaml
        - repoURL: https://github.com/ably77/agw-federated-infra.git
          targetRevision: main
          ref: values
      destination:
        server: '{{server}}'
        namespace: istio-system
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - ServerSideApply=true
---
# Istio CNI (wave 2)
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: istio-cni
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            agw-role: leaf
  template:
    metadata:
      name: 'istio-cni-{{name}}'
      annotations:
        argocd.argoproj.io/sync-wave: "2"
    spec:
      project: platform
      sources:
        - repoURL: us-docker.pkg.dev/soloio-img/istio-helm
          chart: cni
          targetRevision: 1.29.0-solo
          helm:
            valueFiles:
              - $values/helm-apps/istio/cni-values.yaml
        - repoURL: https://github.com/ably77/agw-federated-infra.git
          targetRevision: main
          ref: values
      destination:
        server: '{{server}}'
        namespace: istio-system
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
---
# istiod (wave 3)
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: istiod
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            agw-role: leaf
  template:
    metadata:
      name: 'istiod-{{name}}'
      annotations:
        argocd.argoproj.io/sync-wave: "3"
    spec:
      project: platform
      sources:
        - repoURL: us-docker.pkg.dev/soloio-img/istio-helm
          chart: istiod
          targetRevision: 1.29.0-solo
          helm:
            valueFiles:
              - $values/helm-apps/istio/istiod-values.yaml
              - $values/helm-apps/istio/overlays/{{name}}-istiod-values.yaml
        - repoURL: https://github.com/ably77/agw-federated-infra.git
          targetRevision: main
          ref: values
      destination:
        server: '{{server}}'
        namespace: istio-system
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
---
# ztunnel (wave 4)
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: ztunnel
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            agw-role: leaf
  template:
    metadata:
      name: 'ztunnel-{{name}}'
      annotations:
        argocd.argoproj.io/sync-wave: "4"
    spec:
      project: platform
      sources:
        - repoURL: us-docker.pkg.dev/soloio-img/istio-helm
          chart: ztunnel
          targetRevision: 1.29.0-solo
          helm:
            valueFiles:
              - $values/helm-apps/istio/ztunnel-values.yaml
        - repoURL: https://github.com/ably77/agw-federated-infra.git
          targetRevision: main
          ref: values
      destination:
        server: '{{server}}'
        namespace: istio-system
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

- [ ] **Step 2: Create Agentgateway ApplicationSet**

Create `agw-federated-infra/argocd/applicationsets/agentgateway.yaml`:

```yaml
# Enterprise Agentgateway -- CRDs then controller
---
# AGW CRDs (wave 5)
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: enterprise-agw-crds
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            agw-role: leaf
  template:
    metadata:
      name: 'enterprise-agw-crds-{{name}}'
      annotations:
        argocd.argoproj.io/sync-wave: "5"
    spec:
      project: platform
      sources:
        - repoURL: us-docker.pkg.dev/solo-public/enterprise-agentgateway/charts
          chart: enterprise-agentgateway-crds
          targetRevision: v2.3.0
          helm:
            valueFiles:
              - $values/helm-apps/enterprise-agentgateway/crds-values.yaml
        - repoURL: https://github.com/ably77/agw-federated-infra.git
          targetRevision: main
          ref: values
      destination:
        server: '{{server}}'
        namespace: agentgateway-system
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - ServerSideApply=true
---
# AGW Controller (wave 6)
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: enterprise-agw
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            agw-role: leaf
  template:
    metadata:
      name: 'enterprise-agw-{{name}}'
      annotations:
        argocd.argoproj.io/sync-wave: "6"
    spec:
      project: platform
      sources:
        - repoURL: us-docker.pkg.dev/solo-public/enterprise-agentgateway/charts
          chart: enterprise-agentgateway
          targetRevision: v2.3.0
          helm:
            valueFiles:
              - $values/helm-apps/enterprise-agentgateway/controller-values.yaml
        - repoURL: https://github.com/ably77/agw-federated-infra.git
          targetRevision: main
          ref: values
      destination:
        server: '{{server}}'
        namespace: agentgateway-system
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

- [ ] **Step 3: Create Enforcement ApplicationSet**

Create `agw-federated-infra/argocd/applicationsets/enforcement.yaml`:

```yaml
# Kyverno admission controller (wave 7)
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: kyverno
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            agw-role: leaf
  template:
    metadata:
      name: 'kyverno-{{name}}'
      annotations:
        argocd.argoproj.io/sync-wave: "7"
    spec:
      project: platform
      sources:
        - repoURL: https://kyverno.github.io/kyverno/
          chart: kyverno
          targetRevision: 3.*
          helm:
            valueFiles:
              - $values/helm-apps/kyverno/values.yaml
        - repoURL: https://github.com/ably77/agw-federated-infra.git
          targetRevision: main
          ref: values
      destination:
        server: '{{server}}'
        namespace: kyverno
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

- [ ] **Step 4: Create Monitoring ApplicationSet**

Create `agw-federated-infra/argocd/applicationsets/monitoring.yaml`:

```yaml
# Prometheus + Grafana (wave 8)
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: kube-prometheus
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            agw-role: leaf
  template:
    metadata:
      name: 'kube-prometheus-{{name}}'
      annotations:
        argocd.argoproj.io/sync-wave: "8"
    spec:
      project: platform
      sources:
        - repoURL: https://prometheus-community.github.io/helm-charts
          chart: kube-prometheus-stack
          targetRevision: 80.*
          helm:
            valueFiles:
              - $values/helm-apps/monitoring/kube-prometheus-stack-values.yaml
        - repoURL: https://github.com/ably77/agw-federated-infra.git
          targetRevision: main
          ref: values
      destination:
        server: '{{server}}'
        namespace: monitoring
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - ServerSideApply=true
```

- [ ] **Step 5: Create Infra Config ApplicationSet**

Create `agw-federated-infra/argocd/applicationsets/infra-config.yaml`:

```yaml
# Kustomize infra config -- backends, policies, mesh, admission, observability (wave 10)
# Uses cluster generator; overlay path derived from cluster name label
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: agw-infra-config
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            agw-role: leaf
        values:
          overlay: '{{metadata.labels.agw-leaf-name}}'
  template:
    metadata:
      name: 'agw-infra-config-{{name}}'
      annotations:
        argocd.argoproj.io/sync-wave: "10"
    spec:
      project: platform
      source:
        repoURL: https://github.com/ably77/agw-federated-infra.git
        targetRevision: main
        path: 'overlays/{{values.overlay}}'
      destination:
        server: '{{server}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - ServerSideApply=true
```

- [ ] **Step 6: Create Cluster Apps ApplicationSet**

Create `agw-federated-infra/argocd/applicationsets/cluster-apps.yaml`:

```yaml
# Developer cluster repos -- workloads and routes (wave 20)
# Each leaf cluster has a dedicated repo; the repo URL is stored as a cluster label
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: agw-cluster-apps
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            agw-role: leaf
        values:
          clusterRepo: '{{metadata.labels.agw-cluster-repo}}'
  template:
    metadata:
      name: 'agw-cluster-{{name}}'
      annotations:
        argocd.argoproj.io/sync-wave: "20"
    spec:
      project: developers
      source:
        repoURL: '{{values.clusterRepo}}'
        targetRevision: main
        path: '.'
      destination:
        server: '{{server}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

- [ ] **Step 7: Commit**

```bash
cd /Users/alexly-solo/Desktop/solo/solo-github/agentgateway-gitops/agw-federated-infra
git add argocd/applicationsets/
git commit -m "feat: add ApplicationSets for all product groups and config"
```

---

## Task 4: Infra Repo -- Helm Values

**Files:**
- Create: all files under `agw-federated-infra/helm-apps/`

- [ ] **Step 1: Create Gateway API CRDs values**

Create `agw-federated-infra/helm-apps/gateway-api-crds/base-values.yaml`:

```yaml
# Gateway API CRDs are applied via raw directory, not Helm.
# This file exists for documentation -- the actual version is pinned
# in the ApplicationSet targetRevision (v1.5.0).
{}
```

- [ ] **Step 2: Create Istio base values**

Create `agw-federated-infra/helm-apps/istio/base-values.yaml`:

```yaml
# Shared across all Istio components
global:
  hub: us-docker.pkg.dev/soloio-img/istio
  tag: 1.29.0-solo
  variant: distroless
```

- [ ] **Step 3: Create Istio CNI values**

Create `agw-federated-infra/helm-apps/istio/cni-values.yaml`:

```yaml
profile: ambient
ambient:
  dnsCapture: true
excludeNamespaces:
  - istio-system
  - kube-system
global:
  hub: us-docker.pkg.dev/soloio-img/istio
  tag: 1.29.0-solo
  variant: distroless
```

- [ ] **Step 4: Create istiod values**

Create `agw-federated-infra/helm-apps/istio/istiod-values.yaml`:

```yaml
profile: ambient
global:
  hub: us-docker.pkg.dev/soloio-img/istio
  tag: 1.29.0-solo
  variant: distroless
env:
  PILOT_ENABLE_IP_AUTOALLOCATE: "true"
  PILOT_ENABLE_K8S_SELECT_WORKLOAD_ENTRIES: "false"
  PILOT_SKIP_VALIDATE_TRUST_DOMAIN: "true"
platforms:
  peering:
    enabled: true
# license.value is set via --helm-set by the install script
# or via a sealed secret in production
```

- [ ] **Step 5: Create ztunnel values**

Create `agw-federated-infra/helm-apps/istio/ztunnel-values.yaml`:

```yaml
profile: ambient
logLevel: info
global:
  hub: us-docker.pkg.dev/soloio-img/istio
  tag: 1.29.0-solo
  variant: distroless
resources:
  requests:
    cpu: 500m
    memory: 2048Mi
istioNamespace: istio-system
env:
  L7_ENABLED: "true"
  SKIP_VALIDATE_TRUST_DOMAIN: "true"
```

- [ ] **Step 6: Create Istio per-leaf overlays**

Create `agw-federated-infra/helm-apps/istio/overlays/leaf-1-istiod-values.yaml`:

```yaml
global:
  multiCluster:
    clusterName: cluster2
  network: cluster2
meshConfig:
  trustDomain: cluster2.local
```

Create `agw-federated-infra/helm-apps/istio/overlays/leaf-2-istiod-values.yaml`:

```yaml
global:
  multiCluster:
    clusterName: cluster3
  network: cluster3
meshConfig:
  trustDomain: cluster3.local
```

- [ ] **Step 7: Create AGW CRDs values**

Create `agw-federated-infra/helm-apps/enterprise-agentgateway/crds-values.yaml`:

```yaml
# Enterprise Agentgateway CRDs -- no special values needed
{}
```

- [ ] **Step 8: Create AGW controller values**

Create `agw-federated-infra/helm-apps/enterprise-agentgateway/controller-values.yaml`:

```yaml
# license key is set via --helm-set by the install script
gatewayClassParametersRefs:
  enterprise-agentgateway:
    group: enterpriseagentgateway.solo.io
    kind: EnterpriseAgentgatewayParameters
    name: agentgateway-config
    namespace: agentgateway-system
```

- [ ] **Step 9: Create Kyverno values**

Create `agw-federated-infra/helm-apps/kyverno/values.yaml`:

```yaml
admissionController:
  replicas: 1
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi
backgroundController:
  replicas: 1
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
cleanupController:
  replicas: 1
reportsController:
  replicas: 1
```

- [ ] **Step 10: Create Prometheus/Grafana values**

Create `agw-federated-infra/helm-apps/monitoring/kube-prometheus-stack-values.yaml`:

```yaml
alertmanager:
  enabled: false
grafana:
  adminPassword: "prom-operator"
  service:
    type: ClusterIP
    port: 3000
  sidecar:
    dashboards:
      enabled: true
      label: grafana_dashboard
      labelValue: "1"
      searchNamespace: monitoring
nodeExporter:
  enabled: false
prometheus:
  prometheusSpec:
    ruleSelectorNilUsesHelmValues: false
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
```

- [ ] **Step 11: Commit**

```bash
cd /Users/alexly-solo/Desktop/solo/solo-github/agentgateway-gitops/agw-federated-infra
git add helm-apps/
git commit -m "feat: add Helm values for all products (Istio, AGW, Kyverno, monitoring)"
```

---

## Task 5: Infra Repo -- Kustomize Base (Namespaces, AGW Config, Backends)

**Files:**
- Create: `base/kustomization.yaml`, `base/namespaces.yaml`
- Create: `base/agw-config/` (4 files)
- Create: `base/backends/` (3 files)

- [ ] **Step 1: Create root kustomization**

Create `agw-federated-infra/base/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespaces.yaml
  - agw-config
  - backends
  - global-policies
  - mesh
  - ingress
  - admission-policies
  - rbac
  - secrets
  - observability
```

- [ ] **Step 2: Create namespaces**

Create `agw-federated-infra/base/namespaces.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: wgu-demo
  labels:
    istio.io/dataplane-mode: ambient
---
apiVersion: v1
kind: Namespace
metadata:
  name: wgu-demo-frontend
  labels:
    istio.io/dataplane-mode: ambient
```

- [ ] **Step 3: Create AGW config kustomization**

Create `agw-federated-infra/base/agw-config/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - gateway-parameters.yaml
  - gateway.yaml
  - access-logging.yaml
  - tracing.yaml
```

- [ ] **Step 4: Create gateway parameters**

Create `agw-federated-infra/base/agw-config/gateway-parameters.yaml`:

```yaml
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayParameters
metadata:
  name: agentgateway-config
  namespace: agentgateway-system
spec:
  sharedExtensions:
    extauth:
      enabled: true
      deployment:
        spec:
          replicas: 1
    ratelimiter:
      enabled: true
      deployment:
        spec:
          replicas: 1
    extCache:
      enabled: true
      deployment:
        spec:
          replicas: 1
  logging:
    level: info
  service:
    spec:
      type: ClusterIP
  deployment:
    spec:
      replicas: 1
      template:
        metadata:
          labels:
            istio.io/dataplane-mode: ambient
```

- [ ] **Step 5: Create gateway**

Create `agw-federated-infra/base/agw-config/gateway.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: agentgateway-proxy
  namespace: agentgateway-system
spec:
  gatewayClassName: enterprise-agentgateway
  listeners:
    - name: http
      port: 8080
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: All
```

- [ ] **Step 6: Create access logging policy**

Create `agw-federated-infra/base/agw-config/access-logging.yaml`:

```yaml
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: access-logs
  namespace: agentgateway-system
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: agentgateway-proxy
  frontend:
    accessLog:
      attributes:
        add:
        - name: llm.prompt
          expression: llm.prompt
        - name: llm.completion
          expression: 'llm.completion[0]'
        - name: llm.streaming
          expression: llm.streaming
```

- [ ] **Step 7: Create tracing policy**

Create `agw-federated-infra/base/agw-config/tracing.yaml`:

```yaml
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: tracing
  namespace: agentgateway-system
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: agentgateway-proxy
  frontend:
    tracing:
      backendRef:
        name: solo-enterprise-telemetry-collector
        namespace: kagent
        port: 4317
      protocol: GRPC
      randomSampling: "true"
      attributes:
        add:
        - name: jwt
          expression: jwt
        - name: response.body
          expression: json(response.body)
```

- [ ] **Step 8: Create backends kustomization**

Create `agw-federated-infra/base/backends/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - openai.yaml
  - mcp-backend.yaml
```

- [ ] **Step 9: Create OpenAI backend**

Create `agw-federated-infra/base/backends/openai.yaml`:

```yaml
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: openai
  namespace: agentgateway-system
spec:
  ai:
    provider:
      openai: {}
  policies:
    auth:
      secretRef:
        name: enrollment-openai-secret
```

- [ ] **Step 10: Create MCP backend**

Create `agw-federated-infra/base/backends/mcp-backend.yaml`:

```yaml
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: financial-aid-mcp-backend
  namespace: agentgateway-system
spec:
  mcp:
    targets:
    - name: financial-aid-mcp-target
      static:
        host: financial-aid-mcp.wgu-demo.svc.cluster.local
        port: 8082
        protocol: StreamableHTTP
```

- [ ] **Step 11: Commit**

```bash
cd /Users/alexly-solo/Desktop/solo/solo-github/agentgateway-gitops/agw-federated-infra
git add base/kustomization.yaml base/namespaces.yaml base/agw-config/ base/backends/
git commit -m "feat: add kustomize base -- namespaces, AGW config, backends"
```

---

## Task 6: Infra Repo -- Kustomize Base (Global Policies, Mesh)

**Files:**
- Create: `base/global-policies/` (3 files)
- Create: `base/mesh/` (8 files)

- [ ] **Step 1: Create global policies kustomization**

Create `agw-federated-infra/base/global-policies/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - guardrails.yaml
  - rate-limit.yaml
```

- [ ] **Step 2: Create guardrails**

Create `agw-federated-infra/base/global-policies/guardrails.yaml`:

```yaml
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: wgu-enrollment-guardrails
  namespace: agentgateway-system
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: wgu-enrollment
  backend:
    ai:
      promptGuard:
        request:
        # PII Detection (FERPA compliance)
        - regex:
            action: Reject
            builtins:
            - CreditCard
            - Ssn
          response:
            message: "Request blocked: personally identifiable information (PII) detected. Do not include SSNs or credit card numbers in prompts. This policy enforces FERPA compliance."
            statusCode: 422
        # Prompt Injection Protection
        - regex:
            action: Reject
            matches:
            - "(?i)(ignore|disregard|forget|override|bypass)\\s+(all\\s+|any\\s+|your\\s+)?(previous|prior|earlier|above|existing)\\s+(instructions|rules|guidelines|directives)"
            - "(?i)(you are now|from now on you are|henceforth you are)\\s+(a |an |the )?(unrestricted|unfiltered|uncensored|DAN)"
            - "(?i)(do anything now|DAN mode|enable DAN|activate DAN)"
          response:
            message: "Request blocked: prompt injection attempt detected."
            statusCode: 403
        # Credentials & Secrets
        - regex:
            action: Reject
            matches:
            - "\\bAKIA[0-9A-Z]{16}\\b"
            - "\\bsk-[a-zA-Z0-9_-]{20,}\\b"
            - "-----BEGIN\\s+(RSA\\s+|EC\\s+)?PRIVATE KEY-----"
            - "(?i)(password|secret|token|api[_-]?key)\\s*[=:]\\s*[\"']?[^\\s\"']{8,}"
          response:
            message: "Request blocked: credential or secret detected in prompt."
            statusCode: 422
        # Response: mask PII in LLM output
        response:
        - regex:
            action: Mask
            builtins:
            - CreditCard
            - Ssn
            - PhoneNumber
          response:
            message: "Response filtered: PII redacted from model output."
```

- [ ] **Step 3: Create rate limit**

Create `agw-federated-infra/base/global-policies/rate-limit.yaml`:

```yaml
apiVersion: ratelimit.solo.io/v1alpha1
kind: RateLimitConfig
metadata:
  name: wgu-enrollment-token-limit
  namespace: agentgateway-system
spec:
  raw:
    descriptors:
    - key: generic_key
      value: wgu-enrollment
      rateLimit:
        requestsPerUnit: 100000
        unit: HOUR
    rateLimits:
    - actions:
      - genericKey:
          descriptorValue: wgu-enrollment
      type: TOKEN
---
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: wgu-enrollment-rate-limit
  namespace: agentgateway-system
spec:
  targetRefs:
  - name: wgu-enrollment
    group: gateway.networking.k8s.io
    kind: HTTPRoute
  traffic:
    entRateLimit:
      global:
        rateLimitConfigRefs:
        - name: wgu-enrollment-token-limit
```

- [ ] **Step 4: Create mesh kustomization**

Create `agw-federated-infra/base/mesh/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deny-all.yaml
  - waypoint.yaml
  - chatbot-to-data-product.yaml
  - data-product-to-graphdb.yaml
  - gateway-to-financial-aid.yaml
  - ingress-to-chatbot.yaml
  - waypoint-to-backends.yaml
```

- [ ] **Step 5: Create deny-all baseline**

Create `agw-federated-infra/base/mesh/deny-all.yaml`:

```yaml
# Deny-all baseline: no service can talk to anything unless explicitly allowed.
# This is the FERPA boundary -- student data is locked down by default.
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: wgu-demo
spec:
  {}
---
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: wgu-demo-frontend
spec:
  {}
```

- [ ] **Step 6: Create waypoint + telemetry**

Create `agw-federated-infra/base/mesh/waypoint.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: wgu-demo-waypoint
  namespace: wgu-demo
spec:
  gatewayClassName: istio-waypoint
  listeners:
  - name: mesh
    port: 15008
    protocol: HBONE
    allowedRoutes:
      namespaces:
        from: All
---
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: wgu-demo-access-logging
  namespace: wgu-demo
spec:
  targetRefs:
  - kind: Gateway
    group: gateway.networking.k8s.io
    name: wgu-demo-waypoint
  accessLogging:
  - providers:
    - name: envoy
```

- [ ] **Step 7: Create mesh authorization policies**

Create `agw-federated-infra/base/mesh/chatbot-to-data-product.yaml`:

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: chatbot-to-data-product
  namespace: wgu-demo
spec:
  targetRefs:
  - kind: Service
    group: ""
    name: data-product-api
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - "*/ns/wgu-demo-frontend/sa/enrollment-chatbot"
```

Create `agw-federated-infra/base/mesh/data-product-to-graphdb.yaml`:

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: data-product-to-graphdb
  namespace: wgu-demo
spec:
  targetRefs:
  - kind: Service
    group: ""
    name: graph-db-mock
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - "*/ns/wgu-demo/sa/data-product-api"
```

Create `agw-federated-infra/base/mesh/gateway-to-financial-aid.yaml`:

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: gateway-to-financial-aid
  namespace: wgu-demo
spec:
  targetRefs:
  - kind: Service
    group: ""
    name: financial-aid-mcp
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - "*/ns/agentgateway-system/sa/agentgateway-proxy"
```

Create `agw-federated-infra/base/mesh/ingress-to-chatbot.yaml`:

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: ingress-to-chatbot
  namespace: wgu-demo-frontend
spec:
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - "*/ns/agentgateway-system/sa/ingress"
    to:
    - operation:
        ports: ["8501"]
```

Create `agw-federated-infra/base/mesh/waypoint-to-backends.yaml`:

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: waypoint-to-backends
  namespace: wgu-demo
spec:
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - "*/ns/wgu-demo/sa/wgu-demo-waypoint"
```

- [ ] **Step 8: Commit**

```bash
cd /Users/alexly-solo/Desktop/solo/solo-github/agentgateway-gitops/agw-federated-infra
git add base/global-policies/ base/mesh/
git commit -m "feat: add kustomize base -- global policies, mesh authorization"
```

---

## Task 7: Infra Repo -- Kustomize Base (Ingress, Admission, RBAC, Secrets, Observability)

**Files:**
- Create: `base/ingress/` (3 files)
- Create: `base/admission-policies/` (4 files)
- Create: `base/rbac/` (3 files)
- Create: `base/secrets/` (2 files)
- Create: `base/observability/` (3 files)

- [ ] **Step 1: Create ingress kustomization and resources**

Create `agw-federated-infra/base/ingress/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ingress-parameters.yaml
  - ingress-gateway.yaml
```

Create `agw-federated-infra/base/ingress/ingress-parameters.yaml`:

```yaml
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayParameters
metadata:
  name: ingress-agentgateway-config
  namespace: agentgateway-system
spec:
  logging:
    level: info
  service:
    spec:
      type: ClusterIP
  deployment:
    spec:
      replicas: 1
      template:
        metadata:
          labels:
            istio.io/dataplane-mode: ambient
        spec:
          containers:
          - name: agentgateway
            resources:
              requests:
                cpu: 50m
                memory: 64Mi
```

Create `agw-federated-infra/base/ingress/ingress-gateway.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: ingress
  namespace: agentgateway-system
spec:
  gatewayClassName: enterprise-agentgateway
  infrastructure:
    parametersRef:
      group: enterpriseagentgateway.solo.io
      kind: EnterpriseAgentgatewayParameters
      name: ingress-agentgateway-config
  listeners:
  - allowedRoutes:
      namespaces:
        from: All
    name: http
    port: 80
    protocol: HTTP
```

- [ ] **Step 2: Create admission policies**

Create `agw-federated-infra/base/admission-policies/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deny-local-backends.yaml
  - deny-policy-override.yaml
  - require-guardrail-ref.yaml
```

Create `agw-federated-infra/base/admission-policies/deny-local-backends.yaml`:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: deny-local-agentgateway-backends
  annotations:
    policies.kyverno.io/description: >-
      Only the platform team (via GitOps) can create backend resources.
      Ensures a single source of truth for the AI model registry.
spec:
  validationFailureAction: Enforce
  background: false
  rules:
    - name: deny-manual-backend-creation
      match:
        any:
          - resources:
              kinds:
                - AgentgatewayBackend
      exclude:
        any:
          - subjects:
              - kind: ServiceAccount
                name: argocd-application-controller
                namespace: argocd
          - subjects:
              - kind: ServiceAccount
                name: argocd-server
                namespace: argocd
      validate:
        message: >-
          Backend resources can only be managed via the infra git repo.
          Contact the platform team to add a new AI backend.
        deny: {}
```

Create `agw-federated-infra/base/admission-policies/deny-policy-override.yaml`:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: deny-agentgateway-policy-override
  annotations:
    policies.kyverno.io/description: >-
      Prevents developers from creating or modifying EnterpriseAgentgatewayPolicy
      resources in the platform namespace (agentgateway-system).
spec:
  validationFailureAction: Enforce
  background: false
  rules:
    - name: deny-local-policy-in-platform-ns
      match:
        any:
          - resources:
              kinds:
                - EnterpriseAgentgatewayPolicy
              namespaces:
                - agentgateway-system
      exclude:
        any:
          - subjects:
              - kind: ServiceAccount
                name: argocd-application-controller
                namespace: argocd
          - subjects:
              - kind: ServiceAccount
                name: argocd-server
                namespace: argocd
      validate:
        message: >-
          Policy resources in the agentgateway-system namespace can only be
          managed via the infra git repo.
        deny: {}
```

Create `agw-federated-infra/base/admission-policies/require-guardrail-ref.yaml`:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-guardrail-reference
  annotations:
    policies.kyverno.io/description: >-
      All AI HTTPRoutes that reference AgentgatewayBackend resources must have
      a corresponding guardrail policy. This is advisory -- the policy logs
      a warning but does not block, as the guardrail association is done via
      EnterpriseAgentgatewayPolicy targetRefs, not inline on the HTTPRoute.
spec:
  validationFailureAction: Audit
  background: true
  rules:
    - name: httproute-backend-ref-audit
      match:
        any:
          - resources:
              kinds:
                - HTTPRoute
      preconditions:
        all:
          - key: "{{ request.object.spec.rules[].backendRefs[].group || '' }}"
            operator: AnyIn
            value:
              - agentgateway.dev
      validate:
        message: >-
          This HTTPRoute references an AgentgatewayBackend. Ensure a matching
          EnterpriseAgentgatewayPolicy with guardrail configuration targets
          this route.
        deny: {}
```

- [ ] **Step 3: Create RBAC**

Create `agw-federated-infra/base/rbac/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - platform-admin-role.yaml
  - developer-role.yaml
```

Create `agw-federated-infra/base/rbac/platform-admin-role.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: agw-platform-admin
rules:
- apiGroups: ["agentgateway.dev"]
  resources: ["agentgatewaybackends"]
  verbs: ["*"]
- apiGroups: ["enterpriseagentgateway.solo.io"]
  resources: ["enterpriseagentgatewaypolicies", "enterpriseagentgatewayparameters"]
  verbs: ["*"]
- apiGroups: ["gateway.networking.k8s.io"]
  resources: ["gateways", "gatewayclasses"]
  verbs: ["*"]
- apiGroups: ["kyverno.io"]
  resources: ["clusterpolicies", "policies"]
  verbs: ["*"]
- apiGroups: ["security.istio.io"]
  resources: ["authorizationpolicies"]
  verbs: ["*"]
```

Create `agw-federated-infra/base/rbac/developer-role.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: agw-developer
rules:
- apiGroups: ["gateway.networking.k8s.io"]
  resources: ["httproutes", "grpcroutes", "listenersets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["gateway.networking.k8s.io"]
  resources: ["referencegrants"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["enterpriseagentgateway.solo.io"]
  resources: ["enterpriseagentgatewaypolicies"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["services", "pods", "configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

- [ ] **Step 4: Create secrets placeholder**

Create `agw-federated-infra/base/secrets/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - external-secrets.yaml
```

Create `agw-federated-infra/base/secrets/external-secrets.yaml`:

```yaml
# ExternalSecret placeholder -- demonstrates the pattern for production use.
# In this reference implementation, the actual secret is created by the install
# script via kubectl create secret. In production, use ExternalSecrets Operator
# with Vault, AWS Secrets Manager, or GCP Secret Manager.
#
# Uncomment and install external-secrets operator to use:
#
# apiVersion: external-secrets.io/v1beta1
# kind: ExternalSecret
# metadata:
#   name: openai-api-key
#   namespace: agentgateway-system
# spec:
#   refreshInterval: 1h
#   secretStoreRef:
#     name: vault-backend
#     kind: ClusterSecretStore
#   target:
#     name: enrollment-openai-secret
#   data:
#     - secretKey: Authorization
#       remoteRef:
#         key: ai-gateway/openai
#         property: bearer-token
---
# Placeholder ConfigMap to keep kustomization valid when ExternalSecret is commented out
apiVersion: v1
kind: ConfigMap
metadata:
  name: secret-management-info
  namespace: agentgateway-system
data:
  info: |
    LLM API keys are managed outside of git.
    Install script: scripts/install-argocd.sh creates the secret.
    Production: use ExternalSecrets Operator with your secret store.
```

- [ ] **Step 5: Create observability resources**

Create `agw-federated-infra/base/observability/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - pod-monitor.yaml
  - grafana-dashboard-configmap.yaml
```

Create `agw-federated-infra/base/observability/pod-monitor.yaml`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: agentgateway-metrics
  namespace: agentgateway-system
spec:
  namespaceSelector:
    matchNames:
      - agentgateway-system
  podMetricsEndpoints:
    - port: metrics
  selector:
    matchLabels:
      app.kubernetes.io/name: agentgateway-proxy
```

Create `agw-federated-infra/base/observability/grafana-dashboard-configmap.yaml`:

```yaml
# Grafana dashboard ConfigMap -- the actual JSON is large.
# In the reference implementation, the install script imports it from the
# enrollment-agent repo. For GitOps, embed the JSON in a ConfigMap:
apiVersion: v1
kind: ConfigMap
metadata:
  name: agentgateway-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  agentgateway-overview.json: |
    {
      "annotations": { "list": [] },
      "title": "Agentgateway Overview",
      "uid": "agw-overview",
      "version": 1,
      "panels": [],
      "schemaVersion": 39,
      "templating": { "list": [] },
      "time": { "from": "now-1h", "to": "now" }
    }
```

- [ ] **Step 6: Commit**

```bash
cd /Users/alexly-solo/Desktop/solo/solo-github/agentgateway-gitops/agw-federated-infra
git add base/ingress/ base/admission-policies/ base/rbac/ base/secrets/ base/observability/
git commit -m "feat: add kustomize base -- ingress, admission, RBAC, secrets, observability"
```

---

## Task 8: Infra Repo -- Kustomize Overlays

**Files:**
- Create: `overlays/leaf-1/kustomization.yaml`, `overlays/leaf-1/patches/backend-endpoints.yaml`
- Create: `overlays/leaf-2/kustomization.yaml`, `overlays/leaf-2/patches/backend-endpoints.yaml`

- [ ] **Step 1: Create leaf-1 overlay**

Create `agw-federated-infra/overlays/leaf-1/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patches:
  - path: patches/backend-endpoints.yaml
```

Create `agw-federated-infra/overlays/leaf-1/patches/backend-endpoints.yaml`:

```yaml
# Leaf-1 specific patches
# In production, this would patch secret references to cluster-specific secret stores.
# For the reference implementation with identical clusters, this is minimal.
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: openai
  namespace: agentgateway-system
spec:
  policies:
    auth:
      secretRef:
        name: enrollment-openai-secret
```

- [ ] **Step 2: Create leaf-2 overlay**

Create `agw-federated-infra/overlays/leaf-2/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patches:
  - path: patches/backend-endpoints.yaml
```

Create `agw-federated-infra/overlays/leaf-2/patches/backend-endpoints.yaml`:

```yaml
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: openai
  namespace: agentgateway-system
spec:
  policies:
    auth:
      secretRef:
        name: enrollment-openai-secret
```

- [ ] **Step 3: Verify kustomize builds**

```bash
cd /Users/alexly-solo/Desktop/solo/solo-github/agentgateway-gitops/agw-federated-infra
kustomize build overlays/leaf-1 > /dev/null && echo "leaf-1 overlay: OK"
kustomize build overlays/leaf-2 > /dev/null && echo "leaf-2 overlay: OK"
```

Expected: both print OK.

- [ ] **Step 4: Commit**

```bash
cd /Users/alexly-solo/Desktop/solo/solo-github/agentgateway-gitops/agw-federated-infra
git add overlays/
git commit -m "feat: add kustomize overlays for leaf-1 and leaf-2"
```

---

## Task 9: Cluster-1 Repo -- Services

**Files:**
- Create: all files under `agw-federated-cluster-1/team-enrollment/services/`
- Create: `agw-federated-cluster-1/team-enrollment/kustomization.yaml`
- Create: `agw-federated-cluster-1/team-enrollment/services/kustomization.yaml`

- [ ] **Step 1: Create root kustomization**

Create `agw-federated-cluster-1/team-enrollment/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - services
  - routes
  - policies
```

- [ ] **Step 2: Create services kustomization**

Create `agw-federated-cluster-1/team-enrollment/services/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - enrollment-chatbot.yaml
  - data-product-api.yaml
  - financial-aid-mcp.yaml
  - graph-db-mock.yaml
  - abac-ext-authz.yaml
```

- [ ] **Step 3: Create enrollment chatbot**

Create `agw-federated-cluster-1/team-enrollment/services/enrollment-chatbot.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: enrollment-chatbot
  namespace: wgu-demo-frontend
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: enrollment-chatbot
  namespace: wgu-demo-frontend
  labels:
    app: enrollment-chatbot
spec:
  replicas: 1
  selector:
    matchLabels:
      app: enrollment-chatbot
  template:
    metadata:
      labels:
        app: enrollment-chatbot
    spec:
      serviceAccountName: enrollment-chatbot
      containers:
      - name: enrollment-chatbot
        image: ably7/enrollment-chatbot:0.1.7
        imagePullPolicy: Always
        ports:
        - containerPort: 8501
          name: http
        env:
        - name: GATEWAY_IP
          value: "agentgateway-proxy.agentgateway-system.svc.cluster.local"
        - name: GATEWAY_PORT
          value: "8080"
        - name: GATEWAY_PROTOCOL
          value: "http"
        - name: ORG_NAME
          value: "Western Governors University (WGU)"
        - name: ORG_SHORT
          value: "WGU"
        - name: APP_TITLE
          value: "WGU Enrollment Advisor"
        - name: DATA_PRODUCT_URL
          value: "http://data-product-api.wgu-demo.mesh.internal:8080"
        - name: GRAPH_DB_URL
          value: "http://graph-db-mock.wgu-demo.svc.cluster.local:8081"
        - name: NS_BACKEND
          value: "wgu-demo"
        - name: NS_FRONTEND
          value: "wgu-demo-frontend"
        - name: MCP_URL
          value: "http://agentgateway-proxy.agentgateway-system.svc.cluster.local:8080/financial-aid-mcp"
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: enrollment-chatbot
  namespace: wgu-demo-frontend
  labels:
    app: enrollment-chatbot
spec:
  type: ClusterIP
  selector:
    app: enrollment-chatbot
  ports:
  - name: http
    port: 8501
    targetPort: 8501
```

- [ ] **Step 4: Create data product API**

Create `agw-federated-cluster-1/team-enrollment/services/data-product-api.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: data-product-api
  namespace: wgu-demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: data-product-api
  namespace: wgu-demo
  labels:
    app: data-product-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: data-product-api
  template:
    metadata:
      labels:
        app: data-product-api
    spec:
      serviceAccountName: data-product-api
      containers:
      - name: data-product-api
        image: ably7/data-product-api:0.0.1
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: GRAPH_DB_URL
          value: "http://graph-db-mock.wgu-demo.svc.cluster.local:8081"
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: data-product-api
  namespace: wgu-demo
  labels:
    app: data-product-api
    solo.io/service-scope: global
  annotations:
    networking.istio.io/traffic-distribution: PreferNetwork
spec:
  selector:
    app: data-product-api
  ports:
  - name: http
    port: 8080
    targetPort: 8080
```

- [ ] **Step 5: Create financial aid MCP**

Create `agw-federated-cluster-1/team-enrollment/services/financial-aid-mcp.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: financial-aid-mcp
  namespace: wgu-demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: financial-aid-mcp
  namespace: wgu-demo
  labels:
    app: financial-aid-mcp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: financial-aid-mcp
  template:
    metadata:
      labels:
        app: financial-aid-mcp
    spec:
      serviceAccountName: financial-aid-mcp
      containers:
      - name: financial-aid-mcp
        image: ably7/financial-aid-mcp:0.0.2
        imagePullPolicy: Always
        ports:
        - containerPort: 8082
          name: http
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: financial-aid-mcp
  namespace: wgu-demo
  labels:
    app: financial-aid-mcp
spec:
  selector:
    app: financial-aid-mcp
  ports:
  - name: http
    port: 8082
    targetPort: 8082
```

- [ ] **Step 6: Create graph DB mock**

Create `agw-federated-cluster-1/team-enrollment/services/graph-db-mock.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: graph-db-mock
  namespace: wgu-demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: graph-db-mock
  namespace: wgu-demo
  labels:
    app: graph-db-mock
spec:
  replicas: 1
  selector:
    matchLabels:
      app: graph-db-mock
  template:
    metadata:
      labels:
        app: graph-db-mock
    spec:
      serviceAccountName: graph-db-mock
      containers:
      - name: graph-db-mock
        image: ably7/graph-db-mock:0.0.3
        imagePullPolicy: Always
        ports:
        - containerPort: 8081
          name: http
        readinessProbe:
          httpGet:
            path: /health
            port: 8081
          initialDelaySeconds: 5
          periodSeconds: 10
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: graph-db-mock
  namespace: wgu-demo
  labels:
    app: graph-db-mock
spec:
  selector:
    app: graph-db-mock
  ports:
  - name: http
    port: 8081
    targetPort: 8081
```

- [ ] **Step 7: Create ABAC ext-authz**

Create `agw-federated-cluster-1/team-enrollment/services/abac-ext-authz.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: agentgateway-system
  name: abac-ext-authz
  labels:
    app: abac-ext-authz
spec:
  replicas: 1
  selector:
    matchLabels:
      app: abac-ext-authz
  template:
    metadata:
      labels:
        app: abac-ext-authz
        app.kubernetes.io/name: abac-ext-authz
    spec:
      containers:
      - image: ably7/abac-ext-authz:latest
        imagePullPolicy: Always
        name: abac-ext-authz
        ports:
        - containerPort: 9000
        env:
        - name: PORT
          value: "9000"
        - name: ABAC_POLICIES
          value: "enrollment-advisor:standard=gpt-4o-mini,analytics-agent:premium=gpt-4o|gpt-4o-mini"
---
apiVersion: v1
kind: Service
metadata:
  namespace: agentgateway-system
  name: abac-ext-authz
  labels:
    app: abac-ext-authz
spec:
  ports:
  - port: 4444
    targetPort: 9000
    protocol: TCP
    appProtocol: kubernetes.io/h2c
  selector:
    app: abac-ext-authz
```

- [ ] **Step 8: Commit**

```bash
cd /Users/alexly-solo/Desktop/solo/solo-github/agentgateway-gitops/agw-federated-cluster-1
git add team-enrollment/kustomization.yaml team-enrollment/services/
git commit -m "feat: add enrollment team services"
```

---

## Task 10: Cluster-1 Repo -- Routes, Policies, AppProject

**Files:**
- Create: all files under `agw-federated-cluster-1/team-enrollment/routes/`
- Create: all files under `agw-federated-cluster-1/team-enrollment/policies/`
- Create: `agw-federated-cluster-1/argocd/appproject.yaml`

- [ ] **Step 1: Create routes kustomization**

Create `agw-federated-cluster-1/team-enrollment/routes/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - httproute-openai.yaml
  - httproute-mcp.yaml
  - ingress-routes.yaml
  - reference-grant.yaml
```

- [ ] **Step 2: Create OpenAI HTTPRoute**

Create `agw-federated-cluster-1/team-enrollment/routes/httproute-openai.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: wgu-enrollment
  namespace: agentgateway-system
spec:
  parentRefs:
  - name: agentgateway-proxy
    namespace: agentgateway-system
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /openai
    backendRefs:
    - name: openai
      group: agentgateway.dev
      kind: AgentgatewayBackend
    timeouts:
      request: "120s"
```

- [ ] **Step 3: Create MCP HTTPRoute**

Create `agw-federated-cluster-1/team-enrollment/routes/httproute-mcp.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: financial-aid-mcp
  namespace: agentgateway-system
spec:
  parentRefs:
  - name: agentgateway-proxy
    namespace: agentgateway-system
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /financial-aid-mcp
    backendRefs:
    - name: financial-aid-mcp-backend
      group: agentgateway.dev
      kind: AgentgatewayBackend
    timeouts:
      request: "0s"
```

- [ ] **Step 4: Create ingress routes**

Create `agw-federated-cluster-1/team-enrollment/routes/ingress-routes.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: chatbot-ingress-route
  namespace: wgu-demo-frontend
spec:
  hostnames:
  - "enroll.glootest.com"
  parentRefs:
  - name: ingress
    namespace: agentgateway-system
  rules:
  - backendRefs:
    - name: enrollment-chatbot
      port: 8501
    matches:
    - path:
        type: PathPrefix
        value: /
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: grafana-ingress-route
  namespace: monitoring
spec:
  hostnames:
  - "grafana.glootest.com"
  parentRefs:
  - name: ingress
    namespace: agentgateway-system
  rules:
  - backendRefs:
    - name: grafana-prometheus
      port: 3000
    matches:
    - path:
        type: PathPrefix
        value: /
```

- [ ] **Step 5: Create ReferenceGrant**

Create `agw-federated-cluster-1/team-enrollment/routes/reference-grant.yaml`:

```yaml
# Allow HTTPRoutes in agentgateway-system to reference services in wgu-demo
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-agw-to-wgu-demo
  namespace: wgu-demo
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: agentgateway-system
  to:
  - group: ""
    kind: Service
---
# Allow HTTPRoutes in wgu-demo-frontend to reference the ingress gateway
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-frontend-to-agw
  namespace: agentgateway-system
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: wgu-demo-frontend
  to:
  - group: gateway.networking.k8s.io
    kind: Gateway
```

- [ ] **Step 6: Create policies kustomization and chatbot RBAC**

Create `agw-federated-cluster-1/team-enrollment/policies/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - chatbot-rbac.yaml
```

Create `agw-federated-cluster-1/team-enrollment/policies/chatbot-rbac.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: enrollment-chatbot-mesh-demo
rules:
- apiGroups: ["security.istio.io"]
  resources: ["authorizationpolicies"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "patch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "patch"]
- apiGroups: ["apps"]
  resources: ["deployments/scale"]
  verbs: ["get", "update", "patch"]
- apiGroups: ["networking.istio.io"]
  resources: ["serviceentries"]
  verbs: ["get", "list"]
- apiGroups: ["enterpriseagentgateway.solo.io"]
  resources: ["enterpriseagentgatewaypolicies"]
  verbs: ["get", "list", "create", "apply", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: enrollment-chatbot-mesh-demo
subjects:
- kind: ServiceAccount
  name: enrollment-chatbot
  namespace: wgu-demo-frontend
roleRef:
  kind: ClusterRole
  name: enrollment-chatbot-mesh-demo
  apiGroup: rbac.authorization.k8s.io
```

- [ ] **Step 7: Create AppProject**

Create `agw-federated-cluster-1/argocd/appproject.yaml`:

```yaml
# This AppProject is informational -- the actual project used by ArgoCD
# is defined in the infra repo (argocd/projects/developers.yaml).
# This file documents the namespace restrictions for this cluster's workloads.
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: developers-cluster-1
  namespace: argocd
spec:
  description: Developer workloads for leaf-1 (cluster2)
  sourceRepos:
    - https://github.com/ably77/agw-federated-cluster-1.git
  destinations:
    - namespace: wgu-demo
      server: '*'
    - namespace: wgu-demo-frontend
      server: '*'
    - namespace: agentgateway-system
      server: '*'
    - namespace: monitoring
      server: '*'
  clusterResourceWhitelist:
    - group: rbac.authorization.k8s.io
      kind: ClusterRole
    - group: rbac.authorization.k8s.io
      kind: ClusterRoleBinding
```

- [ ] **Step 8: Verify kustomize build**

```bash
cd /Users/alexly-solo/Desktop/solo/solo-github/agentgateway-gitops/agw-federated-cluster-1
kustomize build team-enrollment > /dev/null && echo "cluster-1 build: OK"
```

Expected: OK

- [ ] **Step 9: Commit**

```bash
cd /Users/alexly-solo/Desktop/solo/solo-github/agentgateway-gitops/agw-federated-cluster-1
git add team-enrollment/routes/ team-enrollment/policies/ argocd/
git commit -m "feat: add routes, policies, AppProject for enrollment team"
```

---

## Task 11: Cluster-2 Repo -- Mirror of Cluster-1

**Files:**
- Create: entire `agw-federated-cluster-2/` mirroring cluster-1 structure

- [ ] **Step 1: Copy cluster-1 content to cluster-2**

```bash
cd /Users/alexly-solo/Desktop/solo/solo-github/agentgateway-gitops

# Copy team-enrollment directory
cp -r agw-federated-cluster-1/team-enrollment/ agw-federated-cluster-2/team-enrollment/

# Copy argocd directory
cp -r agw-federated-cluster-1/argocd/ agw-federated-cluster-2/argocd/
```

- [ ] **Step 2: Update cluster-2 AppProject**

Edit `agw-federated-cluster-2/argocd/appproject.yaml` -- change name and description:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: developers-cluster-2
  namespace: argocd
spec:
  description: Developer workloads for leaf-2 (cluster3)
  sourceRepos:
    - https://github.com/ably77/agw-federated-cluster-2.git
  destinations:
    - namespace: wgu-demo
      server: '*'
    - namespace: wgu-demo-frontend
      server: '*'
    - namespace: agentgateway-system
      server: '*'
    - namespace: monitoring
      server: '*'
  clusterResourceWhitelist:
    - group: rbac.authorization.k8s.io
      kind: ClusterRole
    - group: rbac.authorization.k8s.io
      kind: ClusterRoleBinding
```

- [ ] **Step 3: Verify kustomize build**

```bash
cd /Users/alexly-solo/Desktop/solo/solo-github/agentgateway-gitops/agw-federated-cluster-2
kustomize build team-enrollment > /dev/null && echo "cluster-2 build: OK"
```

Expected: OK

- [ ] **Step 4: Commit**

```bash
cd /Users/alexly-solo/Desktop/solo/solo-github/agentgateway-gitops/agw-federated-cluster-2
git add team-enrollment/ argocd/
git commit -m "feat: add enrollment team workloads (mirror of cluster-1)"
```

---

## Task 12: Install Script -- install-argocd.sh

**Files:**
- Create: `agw-federated-infra/scripts/install-argocd.sh`

- [ ] **Step 1: Create the install script**

Create `agw-federated-infra/scripts/install-argocd.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

# AGW Federated GitOps -- ArgoCD Bootstrap Script
# Installs ArgoCD on the hub cluster (cluster1) and registers leaf clusters.
#
# Prerequisites:
#   - 3 Colima clusters running: cluster1 (hub), cluster2 (leaf-1), cluster3 (leaf-2)
#   - SOLO_TRIAL_LICENSE_KEY and OPENAI_API_KEY environment variables set
#   - helm, kubectl, argocd CLI installed
#
# Usage:
#   export SOLO_TRIAL_LICENSE_KEY=<key>
#   export OPENAI_API_KEY=<key>
#   ./scripts/install-argocd.sh

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
REPO_ROOT="$(cd "$SCRIPT_DIR/.." && pwd)"

HUB_CTX="${HUB_CTX:-cluster1}"
LEAF1_CTX="${LEAF1_CTX:-cluster2}"
LEAF2_CTX="${LEAF2_CTX:-cluster3}"

ARGOCD_VERSION="${ARGOCD_VERSION:-7.8.13}"

# =============================================================================
# Validation
# =============================================================================
validate() {
  echo "=== Validating prerequisites ==="

  # Check env vars
  for var in SOLO_TRIAL_LICENSE_KEY OPENAI_API_KEY; do
    if [ -z "${!var:-}" ]; then
      echo "ERROR: $var is not set. Run: export $var=<your-key>"
      exit 1
    fi
  done

  # Check CLIs
  for cmd in helm kubectl argocd colima; do
    if ! command -v "$cmd" &>/dev/null; then
      echo "ERROR: $cmd not found. Please install it."
      exit 1
    fi
  done

  # Check clusters reachable
  for ctx in $HUB_CTX $LEAF1_CTX $LEAF2_CTX; do
    if ! kubectl cluster-info --context "$ctx" &>/dev/null; then
      echo "ERROR: Cannot reach cluster '$ctx'. Is Colima running?"
      echo "  colima start --profile ${ctx#cluster}"
      exit 1
    fi
  done

  echo "All prerequisites met."
}

# =============================================================================
# Install ArgoCD on hub
# =============================================================================
install_argocd() {
  echo "=== Installing ArgoCD on $HUB_CTX ==="

  kubectl create namespace argocd --context "$HUB_CTX" 2>/dev/null || true

  helm repo add argo https://argoproj.github.io/argo-helm 2>/dev/null || true
  helm repo update argo

  helm upgrade --install argocd argo/argo-cd \
    --namespace argocd \
    --version "$ARGOCD_VERSION" \
    --kube-context "$HUB_CTX" \
    --values "$REPO_ROOT/argocd/bootstrap/values.yaml" \
    --wait --timeout 300s

  echo "Waiting for ArgoCD server..."
  kubectl wait --for=condition=available deploy/argocd-server \
    -n argocd --context "$HUB_CTX" --timeout=180s

  echo "ArgoCD installed on $HUB_CTX."
}

# =============================================================================
# Get ArgoCD admin password
# =============================================================================
get_argocd_password() {
  kubectl get secret argocd-initial-admin-secret -n argocd \
    --context "$HUB_CTX" -o jsonpath='{.data.password}' | base64 -d
}

# =============================================================================
# Discover API server address reachable from hub cluster pods
# =============================================================================
get_api_server_for_argocd() {
  local leaf_ctx=$1

  # Strategy 1: Use Colima VM IP + port 6443
  local vm_name="${leaf_ctx}"
  local vm_ip
  vm_ip=$(colima ssh --profile "$vm_name" -- hostname -I 2>/dev/null | awk '{print $1}')
  if [ -n "$vm_ip" ]; then
    # Verify k3s API is reachable on VM IP from hub
    if kubectl exec -n argocd deploy/argocd-server --context "$HUB_CTX" -- \
       wget -q --spider --timeout=3 "https://${vm_ip}:6443" 2>/dev/null; then
      echo "https://${vm_ip}:6443"
      return
    fi
  fi

  # Strategy 2: Use host gateway IP + forwarded port
  # Get the forwarded port from kubeconfig
  local api_server
  api_server=$(kubectl config view -o jsonpath="{.clusters[?(@.name==\"colima-${leaf_ctx}\")].cluster.server}" 2>/dev/null)
  if [ -z "$api_server" ]; then
    # Try without colima- prefix
    api_server=$(kubectl config view -o jsonpath="{.clusters[?(@.name==\"${leaf_ctx}\")].cluster.server}" 2>/dev/null)
  fi
  local port
  port=$(echo "$api_server" | sed 's|.*:\([0-9]*\)$|\1|')

  # Get host IP as seen from inside the hub cluster
  local host_ip
  host_ip=$(kubectl exec -n argocd deploy/argocd-server --context "$HUB_CTX" -- \
    sh -c "getent hosts host.docker.internal 2>/dev/null | awk '{print \$1}'" 2>/dev/null || true)

  if [ -z "$host_ip" ]; then
    # Fallback: try the default gateway
    host_ip=$(kubectl exec -n argocd deploy/argocd-server --context "$HUB_CTX" -- \
      sh -c "ip route | grep default | awk '{print \$3}'" 2>/dev/null || true)
  fi

  if [ -n "$host_ip" ] && [ -n "$port" ]; then
    echo "https://${host_ip}:${port}"
    return
  fi

  # Strategy 3: Fallback -- use the kubeconfig server directly
  # This won't work from inside a pod but is useful for debugging
  echo "$api_server"
}

# =============================================================================
# Register leaf clusters with ArgoCD
# =============================================================================
register_clusters() {
  echo "=== Registering leaf clusters with ArgoCD ==="

  # Get ArgoCD password and login
  local password
  password=$(get_argocd_password)

  # Port-forward ArgoCD for CLI access
  kubectl port-forward svc/argocd-server -n argocd 8443:443 \
    --context "$HUB_CTX" &>/dev/null &
  local pf_pid=$!
  sleep 3

  argocd login localhost:8443 \
    --username admin \
    --password "$password" \
    --insecure \
    --grpc-web

  # Register each leaf cluster
  local leaf_name leaf_ctx
  for pair in "leaf-1:$LEAF1_CTX" "leaf-2:$LEAF2_CTX"; do
    leaf_name="${pair%%:*}"
    leaf_ctx="${pair##*:}"
    local cluster_repo="https://github.com/ably77/agw-federated-cluster-${leaf_name##leaf-}.git"

    echo "Registering $leaf_name ($leaf_ctx)..."

    # Use argocd cluster add which handles ServiceAccount creation
    argocd cluster add "$leaf_ctx" \
      --name "$leaf_name" \
      --label "agw-role=leaf" \
      --label "agw-leaf-name=${leaf_name}" \
      --label "agw-cluster-repo=${cluster_repo}" \
      --yes

    echo "$leaf_name registered."
  done

  # Clean up port-forward
  kill $pf_pid 2>/dev/null || true

  echo "Leaf clusters registered."
}

# =============================================================================
# Apply ArgoCD resources (projects, applicationsets)
# =============================================================================
apply_argocd_resources() {
  echo "=== Applying ArgoCD AppProjects and ApplicationSets ==="

  kubectl apply -f "$REPO_ROOT/argocd/projects/" --context "$HUB_CTX"
  kubectl apply -f "$REPO_ROOT/argocd/applicationsets/" --context "$HUB_CTX"

  echo "ArgoCD resources applied."
}

# =============================================================================
# Create LLM secrets on leaf clusters
# =============================================================================
create_secrets() {
  echo "=== Creating LLM API key secrets on leaf clusters ==="

  for ctx in $LEAF1_CTX $LEAF2_CTX; do
    kubectl create namespace agentgateway-system --context "$ctx" 2>/dev/null || true
    kubectl create secret generic enrollment-openai-secret \
      -n agentgateway-system \
      --from-literal="Authorization=Bearer $OPENAI_API_KEY" \
      --dry-run=client -oyaml | kubectl apply --context "$ctx" -f -
    echo "Secret created on $ctx."
  done
}

# =============================================================================
# Create shared root CA for Istio multi-cluster
# =============================================================================
create_istio_certs() {
  echo "=== Generating shared root CA for Istio ==="
  local WORK_DIR
  WORK_DIR=$(mktemp -d)

  cat > "$WORK_DIR/root-openssl.cnf" <<'CNFEOF'
[ req ]
prompt = no
distinguished_name = dn
x509_extensions = v3_ca
[ dn ]
C  = US
ST = California
L  = San Francisco
O  = MyOrg
OU = MyUnit
CN = root-cert
[ v3_ca ]
basicConstraints = critical, CA:TRUE, pathlen:1
keyUsage = critical, keyCertSign, cRLSign
subjectKeyIdentifier = hash
CNFEOF

  cat > "$WORK_DIR/intermediate-req.cnf" <<'CNFEOF'
[ req ]
prompt = no
distinguished_name = dn
[ dn ]
C  = US
ST = California
L  = San Francisco
O  = MyOrg
OU = MyUnit
CN = istio-intermediate-ca
CNFEOF

  cat > "$WORK_DIR/ca-ext.cnf" <<'CNFEOF'
[v3_ca]
basicConstraints = critical, CA:TRUE, pathlen:0
keyUsage = critical, keyCertSign, cRLSign
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
CNFEOF

  openssl req -x509 -sha256 -nodes -days 3650 \
    -newkey rsa:2048 -keyout "$WORK_DIR/root-key.pem" \
    -out "$WORK_DIR/root-cert.pem" \
    -config "$WORK_DIR/root-openssl.cnf" -extensions v3_ca 2>/dev/null

  openssl req -new -nodes -newkey rsa:2048 \
    -keyout "$WORK_DIR/ca-key.pem" -out "$WORK_DIR/ca.csr" \
    -config "$WORK_DIR/intermediate-req.cnf" 2>/dev/null

  openssl x509 -req -sha256 -days 3650 \
    -in "$WORK_DIR/ca.csr" \
    -CA "$WORK_DIR/root-cert.pem" -CAkey "$WORK_DIR/root-key.pem" \
    -CAcreateserial -out "$WORK_DIR/ca-cert.pem" \
    -extfile "$WORK_DIR/ca-ext.cnf" -extensions v3_ca 2>/dev/null

  cat "$WORK_DIR/ca-cert.pem" "$WORK_DIR/root-cert.pem" > "$WORK_DIR/cert-chain.pem"

  # Install on both leaf clusters
  for ctx in $LEAF1_CTX $LEAF2_CTX; do
    kubectl create namespace istio-system --context "$ctx" 2>/dev/null || true
    kubectl create secret generic cacerts -n istio-system \
      --from-file=ca-cert.pem="$WORK_DIR/ca-cert.pem" \
      --from-file=ca-key.pem="$WORK_DIR/ca-key.pem" \
      --from-file=root-cert.pem="$WORK_DIR/root-cert.pem" \
      --from-file=cert-chain.pem="$WORK_DIR/cert-chain.pem" \
      --context "$ctx" --dry-run=client -oyaml | kubectl apply --context "$ctx" -f -
  done

  rm -rf "$WORK_DIR"
  echo "Istio CA certs installed on leaf clusters."
}

# =============================================================================
# Wait for ArgoCD sync
# =============================================================================
wait_for_sync() {
  echo "=== Waiting for ArgoCD to sync all applications ==="

  local timeout=600
  local interval=15
  local elapsed=0

  while [ $elapsed -lt $timeout ]; do
    local total healthy
    total=$(kubectl get applications -n argocd --context "$HUB_CTX" --no-headers 2>/dev/null | wc -l | tr -d ' ')
    healthy=$(kubectl get applications -n argocd --context "$HUB_CTX" --no-headers 2>/dev/null | grep -c "Healthy.*Synced" || true)

    echo "  Applications: $healthy/$total healthy+synced (${elapsed}s elapsed)"

    if [ "$total" -gt 0 ] && [ "$healthy" -eq "$total" ]; then
      echo "All applications synced and healthy!"
      return 0
    fi

    sleep $interval
    elapsed=$((elapsed + interval))
  done

  echo "WARNING: Not all applications synced within ${timeout}s."
  echo "Check: kubectl get applications -n argocd --context $HUB_CTX"
  return 1
}

# =============================================================================
# Print access info
# =============================================================================
print_access_info() {
  local password
  password=$(get_argocd_password)

  echo ""
  echo "============================================"
  echo "  AGW Federated GitOps -- Install Complete"
  echo "============================================"
  echo ""
  echo "ArgoCD UI:"
  echo "  kubectl port-forward svc/argocd-server -n argocd 8080:443 --context $HUB_CTX"
  echo "  URL: https://localhost:8080"
  echo "  Username: admin"
  echo "  Password: $password"
  echo ""
  echo "Enrollment Chatbot (leaf-1):"
  echo "  kubectl port-forward svc/enrollment-chatbot -n wgu-demo-frontend 8501:8501 --context $LEAF1_CTX"
  echo "  URL: http://localhost:8501"
  echo ""
  echo "Grafana (leaf-1):"
  echo "  kubectl port-forward svc/grafana-prometheus -n monitoring 3000:3000 --context $LEAF1_CTX"
  echo "  URL: http://localhost:3000 (admin / prom-operator)"
  echo ""
  echo "Enrollment Chatbot (leaf-2):"
  echo "  kubectl port-forward svc/enrollment-chatbot -n wgu-demo-frontend 8502:8501 --context $LEAF2_CTX"
  echo "  URL: http://localhost:8502"
  echo ""
  echo "ArgoCD Applications:"
  echo "  kubectl get applications -n argocd --context $HUB_CTX"
  echo ""
}

# =============================================================================
# Main
# =============================================================================
validate
install_argocd
create_istio_certs
create_secrets
register_clusters
apply_argocd_resources
wait_for_sync || true
print_access_info
```

- [ ] **Step 2: Make executable**

```bash
chmod +x /Users/alexly-solo/Desktop/solo/solo-github/agentgateway-gitops/agw-federated-infra/scripts/install-argocd.sh
```

- [ ] **Step 3: Commit**

```bash
cd /Users/alexly-solo/Desktop/solo/solo-github/agentgateway-gitops/agw-federated-infra
git add scripts/install-argocd.sh
git commit -m "feat: add ArgoCD bootstrap install script"
```

---

## Task 13: Scripts -- register-clusters.sh and validate.sh

**Files:**
- Create: `agw-federated-infra/scripts/register-clusters.sh`
- Create: `agw-federated-infra/scripts/validate.sh`

- [ ] **Step 1: Create register-clusters.sh**

Create `agw-federated-infra/scripts/register-clusters.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Re-register leaf clusters with ArgoCD
# Use after Colima restart when VM IPs change.
#
# Usage: ./scripts/register-clusters.sh

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

HUB_CTX="${HUB_CTX:-cluster1}"
LEAF1_CTX="${LEAF1_CTX:-cluster2}"
LEAF2_CTX="${LEAF2_CTX:-cluster3}"

echo "=== Re-registering leaf clusters ==="

# Get ArgoCD password
PASSWORD=$(kubectl get secret argocd-initial-admin-secret -n argocd \
  --context "$HUB_CTX" -o jsonpath='{.data.password}' | base64 -d)

# Port-forward ArgoCD
kubectl port-forward svc/argocd-server -n argocd 8443:443 \
  --context "$HUB_CTX" &>/dev/null &
PF_PID=$!
sleep 3

argocd login localhost:8443 \
  --username admin \
  --password "$PASSWORD" \
  --insecure \
  --grpc-web

# Remove old registrations
for name in leaf-1 leaf-2; do
  argocd cluster rm "$name" 2>/dev/null || true
done

# Re-register
for pair in "leaf-1:$LEAF1_CTX" "leaf-2:$LEAF2_CTX"; do
  leaf_name="${pair%%:*}"
  leaf_ctx="${pair##*:}"
  cluster_repo="https://github.com/ably77/agw-federated-cluster-${leaf_name##leaf-}.git"

  echo "Registering $leaf_name ($leaf_ctx)..."
  argocd cluster add "$leaf_ctx" \
    --name "$leaf_name" \
    --label "agw-role=leaf" \
    --label "agw-leaf-name=${leaf_name}" \
    --label "agw-cluster-repo=${cluster_repo}" \
    --yes
done

kill $PF_PID 2>/dev/null || true
echo "Done. ArgoCD will re-sync automatically."
```

- [ ] **Step 2: Create validate.sh**

Create `agw-federated-infra/scripts/validate.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

# AGW Federated GitOps -- End-to-end Validation
# Validates that the entire reference architecture is working correctly.
#
# Usage: ./scripts/validate.sh

HUB_CTX="${HUB_CTX:-cluster1}"
LEAF1_CTX="${LEAF1_CTX:-cluster2}"
LEAF2_CTX="${LEAF2_CTX:-cluster3}"

PASS=0
FAIL=0
WARN=0

check() {
  local desc=$1
  shift
  if "$@" &>/dev/null; then
    echo "  [PASS] $desc"
    PASS=$((PASS + 1))
  else
    echo "  [FAIL] $desc"
    FAIL=$((FAIL + 1))
  fi
}

check_warn() {
  local desc=$1
  shift
  if "$@" &>/dev/null; then
    echo "  [PASS] $desc"
    PASS=$((PASS + 1))
  else
    echo "  [WARN] $desc"
    WARN=$((WARN + 1))
  fi
}

# =========================================================================
# Phase 1: ArgoCD Health
# =========================================================================
echo ""
echo "=== Phase 1: ArgoCD Health (hub: $HUB_CTX) ==="

check "ArgoCD server running" \
  kubectl get deploy argocd-server -n argocd --context "$HUB_CTX" \
  -o jsonpath='{.status.readyReplicas}' | grep -q "1"

check "ArgoCD application controller running" \
  kubectl get deploy argocd-application-controller -n argocd --context "$HUB_CTX" \
  -o jsonpath='{.status.readyReplicas}' | grep -q "1"

# Count applications
TOTAL_APPS=$(kubectl get applications -n argocd --context "$HUB_CTX" --no-headers 2>/dev/null | wc -l | tr -d ' ')
HEALTHY_APPS=$(kubectl get applications -n argocd --context "$HUB_CTX" --no-headers 2>/dev/null | grep -c "Healthy.*Synced" || true)
echo "  [INFO] Applications: $HEALTHY_APPS/$TOTAL_APPS healthy+synced"

if [ "$TOTAL_APPS" -gt 0 ] && [ "$HEALTHY_APPS" -eq "$TOTAL_APPS" ]; then
  echo "  [PASS] All ArgoCD applications healthy"
  PASS=$((PASS + 1))
else
  echo "  [FAIL] Not all applications healthy"
  FAIL=$((FAIL + 1))
  kubectl get applications -n argocd --context "$HUB_CTX" --no-headers 2>/dev/null | grep -v "Healthy.*Synced" || true
fi

# =========================================================================
# Phase 2: Infrastructure (per leaf cluster)
# =========================================================================
for leaf_ctx in $LEAF1_CTX $LEAF2_CTX; do
  echo ""
  echo "=== Phase 2: Infrastructure ($leaf_ctx) ==="

  check "istiod running" \
    kubectl get pods -n istio-system -l app=istiod --context "$leaf_ctx" \
    --field-selector=status.phase=Running --no-headers | grep -q .

  check "ztunnel running" \
    kubectl get pods -n istio-system -l app=ztunnel --context "$leaf_ctx" \
    --field-selector=status.phase=Running --no-headers | grep -q .

  check "AGW controller running" \
    kubectl get pods -n agentgateway-system \
    -l app.kubernetes.io/name=enterprise-agentgateway --context "$leaf_ctx" \
    --field-selector=status.phase=Running --no-headers | grep -q .

  check "AGW proxy gateway programmed" \
    kubectl get gateway agentgateway-proxy -n agentgateway-system \
    --context "$leaf_ctx" -o jsonpath='{.status.conditions[?(@.type=="Programmed")].status}' | grep -q "True"

  check "Kyverno running" \
    kubectl get pods -n kyverno -l app.kubernetes.io/component=admission-controller \
    --context "$leaf_ctx" --field-selector=status.phase=Running --no-headers | grep -q .

  check "Prometheus running" \
    kubectl get pods -n monitoring -l app.kubernetes.io/name=prometheus \
    --context "$leaf_ctx" --field-selector=status.phase=Running --no-headers | grep -q .

  check "Grafana running" \
    kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana \
    --context "$leaf_ctx" --field-selector=status.phase=Running --no-headers | grep -q .

  check "Namespace wgu-demo exists with ambient label" \
    kubectl get ns wgu-demo --context "$leaf_ctx" \
    -o jsonpath='{.metadata.labels.istio\.io/dataplane-mode}' | grep -q "ambient"

  check "Namespace wgu-demo-frontend exists with ambient label" \
    kubectl get ns wgu-demo-frontend --context "$leaf_ctx" \
    -o jsonpath='{.metadata.labels.istio\.io/dataplane-mode}' | grep -q "ambient"
done

# =========================================================================
# Phase 3: Workloads (per leaf cluster)
# =========================================================================
for leaf_ctx in $LEAF1_CTX $LEAF2_CTX; do
  echo ""
  echo "=== Phase 3: Workloads ($leaf_ctx) ==="

  for deploy in graph-db-mock data-product-api financial-aid-mcp; do
    check "$deploy ready" \
      kubectl rollout status deploy/$deploy -n wgu-demo --context "$leaf_ctx" --timeout=10s
  done

  check "enrollment-chatbot ready" \
    kubectl rollout status deploy/enrollment-chatbot -n wgu-demo-frontend --context "$leaf_ctx" --timeout=10s

  check "waypoint running" \
    kubectl get pods -n wgu-demo -l gateway.networking.k8s.io/gateway-name=wgu-demo-waypoint \
    --context "$leaf_ctx" --field-selector=status.phase=Running --no-headers | grep -q .

  check "abac-ext-authz ready" \
    kubectl rollout status deploy/abac-ext-authz -n agentgateway-system --context "$leaf_ctx" --timeout=10s
done

# =========================================================================
# Phase 4: Enforcement (test on leaf-1 only)
# =========================================================================
echo ""
echo "=== Phase 4: Kyverno Enforcement ($LEAF1_CTX) ==="

# Test: rogue backend should be blocked
ROGUE_RESULT=$(kubectl apply --context "$LEAF1_CTX" --dry-run=server -f - 2>&1 <<'EOF' || true
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: rogue-backend
  namespace: agentgateway-system
spec:
  ai:
    provider:
      openai: {}
EOF
)
if echo "$ROGUE_RESULT" | grep -qi "denied\|blocked\|validate"; then
  echo "  [PASS] Kyverno blocks rogue AgentgatewayBackend creation"
  PASS=$((PASS + 1))
else
  echo "  [FAIL] Kyverno did NOT block rogue backend: $ROGUE_RESULT"
  FAIL=$((FAIL + 1))
fi

# Test: rogue policy in agentgateway-system should be blocked
POLICY_RESULT=$(kubectl apply --context "$LEAF1_CTX" --dry-run=server -f - 2>&1 <<'EOF' || true
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: rogue-policy
  namespace: agentgateway-system
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: agentgateway-proxy
  backend:
    ai:
      promptGuard: {}
EOF
)
if echo "$POLICY_RESULT" | grep -qi "denied\|blocked\|validate"; then
  echo "  [PASS] Kyverno blocks rogue policy override"
  PASS=$((PASS + 1))
else
  echo "  [FAIL] Kyverno did NOT block rogue policy: $POLICY_RESULT"
  FAIL=$((FAIL + 1))
fi

# =========================================================================
# Phase 5: Traffic (via port-forward on leaf-1)
# =========================================================================
echo ""
echo "=== Phase 5: Traffic Validation ($LEAF1_CTX) ==="

# Port-forward chatbot
kubectl port-forward svc/enrollment-chatbot -n wgu-demo-frontend 18501:8501 \
  --context "$LEAF1_CTX" &>/dev/null &
PF_CHATBOT=$!
sleep 2

check_warn "Chatbot responds" \
  curl -sf -o /dev/null http://localhost:18501/

kill $PF_CHATBOT 2>/dev/null || true

# =========================================================================
# Phase 6: GitOps Drift (leaf-1)
# =========================================================================
echo ""
echo "=== Phase 6: GitOps Drift Test ($LEAF1_CTX) ==="
echo "  [INFO] Skipping drift test (requires waiting for ArgoCD sync interval)."
echo "  [INFO] To test manually:"
echo "    kubectl delete configmap secret-management-info -n agentgateway-system --context $LEAF1_CTX"
echo "    # Wait ~3 min for ArgoCD selfHeal to recreate it"
echo "    kubectl get configmap secret-management-info -n agentgateway-system --context $LEAF1_CTX"

# =========================================================================
# Summary
# =========================================================================
echo ""
echo "============================================"
echo "  Validation Summary"
echo "============================================"
echo "  PASS: $PASS"
echo "  FAIL: $FAIL"
echo "  WARN: $WARN"
echo ""

if [ "$FAIL" -gt 0 ]; then
  echo "  RESULT: FAILURES DETECTED"
  exit 1
else
  echo "  RESULT: ALL CHECKS PASSED"
fi
```

- [ ] **Step 3: Make executable**

```bash
chmod +x /Users/alexly-solo/Desktop/solo/solo-github/agentgateway-gitops/agw-federated-infra/scripts/register-clusters.sh
chmod +x /Users/alexly-solo/Desktop/solo/solo-github/agentgateway-gitops/agw-federated-infra/scripts/validate.sh
```

- [ ] **Step 4: Commit**

```bash
cd /Users/alexly-solo/Desktop/solo/solo-github/agentgateway-gitops/agw-federated-infra
git add scripts/register-clusters.sh scripts/validate.sh
git commit -m "feat: add cluster registration and validation scripts"
```

---

## Task 14: READMEs and Push to GitHub

**Files:**
- Create: `agw-federated-infra/README.md`
- Create: `agw-federated-cluster-1/README.md`
- Create: `agw-federated-cluster-2/README.md`

- [ ] **Step 1: Create infra repo README**

Create `agw-federated-infra/README.md`:

```markdown
# AGW Federated GitOps -- Infra Repo

Platform-team owned repository for the [AGW Federated GitOps Reference Architecture](https://github.com/ably77/agentgateway-federated-gitops-ref-arch).

## What This Repo Contains

- **ArgoCD configuration** -- bootstrap, AppProjects, ApplicationSets
- **Helm values** -- Istio, Enterprise Agentgateway, Kyverno, Prometheus/Grafana
- **Kustomize base** -- backends, global policies, guardrails, mesh, ingress, admission policies, RBAC, observability
- **Kustomize overlays** -- per-leaf-cluster patches (secret refs, endpoints)
- **Scripts** -- install, cluster registration, validation

## Quick Start

```bash
# Prerequisites: 3 Colima clusters (cluster1=hub, cluster2=leaf-1, cluster3=leaf-2)
export SOLO_TRIAL_LICENSE_KEY=<key>
export OPENAI_API_KEY=<key>

./scripts/install-argocd.sh

# Validate
./scripts/validate.sh
```

## Architecture

```
cluster1 (hub)          cluster2 (leaf-1)       cluster3 (leaf-2)
┌──────────────┐        ┌──────────────┐        ┌──────────────┐
│   ArgoCD     │───────>│  Istio       │        │  Istio       │
│   AppSets    │───────>│  AGW CP+DP   │        │  AGW CP+DP   │
│              │───────>│  Kyverno     │        │  Kyverno     │
│              │───────>│  Monitoring  │        │  Monitoring  │
│              │───────>│  Workloads   │        │  Workloads   │
└──────────────┘        └──────────────┘        └──────────────┘
```

## Related Repos

- [agw-federated-cluster-1](https://github.com/ably77/agw-federated-cluster-1) -- Developer workloads for leaf-1
- [agw-federated-cluster-2](https://github.com/ably77/agw-federated-cluster-2) -- Developer workloads for leaf-2
- [Reference Architecture](https://github.com/ably77/agentgateway-federated-gitops-ref-arch) -- Design document
```

- [ ] **Step 2: Create cluster-1 README**

Create `agw-federated-cluster-1/README.md`:

```markdown
# AGW Federated GitOps -- Cluster 1 (Leaf-1)

Developer self-service repository for leaf-1 (cluster2) in the [AGW Federated GitOps Reference Architecture](https://github.com/ably77/agentgateway-federated-gitops-ref-arch).

## What Developers Can Do

- **Routes**: HTTPRoute, GRPCRoute for AI traffic routing
- **Services**: Deploy workloads in team namespaces
- **Policies**: Namespace-scoped timeouts, retries, headers
- **ReferenceGrants**: Cross-namespace backend references

## What Developers Cannot Do

- Create or modify AgentgatewayBackend (blocked by Kyverno)
- Override global guardrail policies (blocked by Kyverno)
- Modify admission policies or RBAC (blocked by Kubernetes RBAC)

## Structure

```
team-enrollment/
├── services/     # Deployments, Services, ServiceAccounts
├── routes/       # HTTPRoutes, ReferenceGrants
└── policies/     # Namespace-scoped RBAC
```

## Related Repos

- [agw-federated-infra](https://github.com/ably77/agw-federated-infra) -- Platform infrastructure
- [agw-federated-cluster-2](https://github.com/ably77/agw-federated-cluster-2) -- Leaf-2 workloads
```

- [ ] **Step 3: Create cluster-2 README**

Create `agw-federated-cluster-2/README.md`:

Same as cluster-1 but with "Cluster 2 (Leaf-2)", "leaf-2 (cluster3)", and updated related repo links.

```markdown
# AGW Federated GitOps -- Cluster 2 (Leaf-2)

Developer self-service repository for leaf-2 (cluster3) in the [AGW Federated GitOps Reference Architecture](https://github.com/ably77/agentgateway-federated-gitops-ref-arch).

## What Developers Can Do

- **Routes**: HTTPRoute, GRPCRoute for AI traffic routing
- **Services**: Deploy workloads in team namespaces
- **Policies**: Namespace-scoped timeouts, retries, headers
- **ReferenceGrants**: Cross-namespace backend references

## What Developers Cannot Do

- Create or modify AgentgatewayBackend (blocked by Kyverno)
- Override global guardrail policies (blocked by Kyverno)
- Modify admission policies or RBAC (blocked by Kubernetes RBAC)

## Structure

```
team-enrollment/
├── services/     # Deployments, Services, ServiceAccounts
├── routes/       # HTTPRoutes, ReferenceGrants
└── policies/     # Namespace-scoped RBAC
```

## Related Repos

- [agw-federated-infra](https://github.com/ably77/agw-federated-infra) -- Platform infrastructure
- [agw-federated-cluster-1](https://github.com/ably77/agw-federated-cluster-1) -- Leaf-1 workloads
```

- [ ] **Step 4: Commit and push all repos**

```bash
cd /Users/alexly-solo/Desktop/solo/solo-github/agentgateway-gitops

# Infra repo
cd agw-federated-infra
git add README.md
git commit -m "docs: add README"
git push -u origin main

# Cluster-1 repo
cd ../agw-federated-cluster-1
git add README.md
git commit -m "docs: add README"
git push -u origin main

# Cluster-2 repo
cd ../agw-federated-cluster-2
git add README.md
git commit -m "docs: add README"
git push -u origin main
```

---

## Task Dependency Graph

```
Task 1 (scaffolding)
  ├── Task 2 (ArgoCD bootstrap) → Task 3 (ApplicationSets)
  ├── Task 4 (Helm values)
  ├── Task 5 (base: ns, agw, backends) → Task 6 (base: policies, mesh) → Task 7 (base: ingress, admission, etc.) → Task 8 (overlays)
  ├── Task 9 (cluster-1 services) → Task 10 (cluster-1 routes/policies)
  ├── Task 11 (cluster-2 mirror -- depends on Task 10)
  ├── Task 12 (install script)
  └── Task 13 (register + validate scripts)
Task 14 (READMEs + push -- depends on all above)
```

**Parallelizable groups after Task 1:**
- Group A: Tasks 2, 3 (ArgoCD config)
- Group B: Tasks 4 (Helm values)
- Group C: Tasks 5, 6, 7, 8 (Kustomize base + overlays)
- Group D: Tasks 9, 10, 11 (Cluster repos)
- Group E: Tasks 12, 13 (Scripts)

Groups A-E are independent of each other and can be worked in parallel.
