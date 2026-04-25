# AGW Federated GitOps Reference Architecture -- Implementation Design

**Date:** 2026-04-23
**Status:** Approved
**Source:** `agentgateway-federated-gitops-ref-arch/README.md` (reference architecture)
**Reference Example:** `enrollment-agent/` (working demo with real k8s manifests)

---

## 1. Overview

Implement the federated GitOps reference architecture described in the README as a working, deployable system across 3 Colima clusters. The implementation decomposes the enrollment-agent demo into infra and developer repos, uses ArgoCD ApplicationSets for hub-to-leaf propagation, and includes Kyverno admission policies for runtime enforcement.

### Requirements

- 3 clusters: 1 hub (ArgoCD only), 2 leaf (AGW + Istio + workloads)
- Full decomposition of enrollment-agent resources across infra/cluster repos
- ArgoCD with ApplicationSets using cluster generator + labels
- Helm for third-party product installs (Istio, AGW, Kyverno, monitoring)
- Kustomize for authored configuration (backends, policies, mesh, routes)
- Working Kyverno admission policies (enforce, not placeholder)
- Install script for ArgoCD bootstrap on local Colima clusters
- Validation script for end-to-end verification

---

## 2. Cluster Topology

| Colima Profile | Context | Role | Resources |
|---|---|---|---|
| `cluster1` | `cluster1` | Hub -- ArgoCD management plane | 2 CPU, 8GB RAM |
| `cluster2` | `cluster2` | Leaf-1 -- AGW + Istio + workloads | 6 CPU, 16GB RAM |
| `cluster3` | `cluster3` | Leaf-2 -- AGW + Istio + workloads | 4 CPU, 16GB RAM |

The hub runs only ArgoCD. No AGW, no Istio, no workloads. Leaf clusters are identical deployment targets managed by ArgoCD ApplicationSets.

---

## 3. Repository Structure

Three GitHub repos under `ably77`:

### 3.1 `agw-federated-infra`

Platform team owns this. Contains everything that must not diverge across leaf clusters.

```
agw-federated-infra/
├── argocd/
│   ├── bootstrap/
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   └── values.yaml                    # ArgoCD Helm values
│   ├── cluster-secrets/
│   │   ├── leaf-1.yaml
│   │   └── leaf-2.yaml
│   ├── projects/
│   │   ├── platform.yaml                  # AppProject for infra
│   │   └── developers.yaml                # AppProject for cluster repos
│   └── applicationsets/
│       ├── istio.yaml                     # Istio components (waves 1-4)
│       ├── agentgateway.yaml              # AGW CRDs + controller (waves 5-6)
│       ├── enforcement.yaml               # Kyverno (wave 7)
│       ├── monitoring.yaml                # Prometheus/Grafana (wave 8)
│       ├── infra-config.yaml              # Kustomize base+overlays (wave 10)
│       └── cluster-apps.yaml              # Developer cluster repos (wave 20)
├── helm-apps/
│   ├── gateway-api-crds/
│   │   └── base-values.yaml
│   ├── istio/
│   │   ├── base-values.yaml
│   │   ├── istiod-values.yaml
│   │   ├── cni-values.yaml
│   │   ├── ztunnel-values.yaml
│   │   └── overlays/
│   │       ├── leaf-1-istiod-values.yaml  # clusterName, network, trustDomain
│   │       └── leaf-2-istiod-values.yaml
│   ├── enterprise-agentgateway/
│   │   ├── crds-values.yaml
│   │   └── controller-values.yaml
│   ├── kyverno/
│   │   └── values.yaml
│   └── monitoring/
│       └── kube-prometheus-stack-values.yaml
├── base/
│   ├── kustomization.yaml
│   ├── namespaces.yaml                    # wgu-demo + wgu-demo-frontend (ambient labeled)
│   ├── agw-config/
│   │   ├── kustomization.yaml
│   │   ├── gateway-parameters.yaml        # EnterpriseAgentgatewayParameters
│   │   ├── gateway.yaml                   # Gateway (agentgateway-proxy)
│   │   ├── access-logging.yaml            # EnterpriseAgentgatewayPolicy
│   │   └── tracing.yaml                   # EnterpriseAgentgatewayPolicy
│   ├── backends/
│   │   ├── kustomization.yaml
│   │   ├── openai.yaml                    # AgentgatewayBackend
│   │   └── mcp-backend.yaml               # AgentgatewayBackend (MCP, resource only)
│   ├── global-policies/
│   │   ├── kustomization.yaml
│   │   ├── guardrails.yaml                # EnterpriseAgentgatewayPolicy (PII, injection)
│   │   └── rate-limit.yaml                # RateLimitConfig + EnterpriseAgentgatewayPolicy
│   ├── mesh/
│   │   ├── kustomization.yaml
│   │   ├── deny-all.yaml                  # AuthorizationPolicy baseline
│   │   ├── waypoint.yaml                  # Gateway (istio-waypoint) + Telemetry
│   │   ├── chatbot-to-data-product.yaml   # AuthorizationPolicy
│   │   ├── data-product-to-graphdb.yaml   # AuthorizationPolicy
│   │   ├── gateway-to-financial-aid.yaml  # AuthorizationPolicy
│   │   ├── ingress-to-chatbot.yaml        # AuthorizationPolicy
│   │   └── waypoint-to-backends.yaml      # AuthorizationPolicy
│   ├── ingress/
│   │   ├── kustomization.yaml
│   │   ├── ingress-gateway.yaml           # Gateway (ingress)
│   │   └── ingress-parameters.yaml        # EnterpriseAgentgatewayParameters
│   ├── admission-policies/
│   │   ├── kustomization.yaml
│   │   ├── deny-local-backends.yaml       # Kyverno ClusterPolicy
│   │   ├── deny-policy-override.yaml      # Kyverno ClusterPolicy
│   │   └── require-guardrail-ref.yaml     # Kyverno ClusterPolicy
│   ├── rbac/
│   │   ├── kustomization.yaml
│   │   ├── platform-admin-role.yaml       # ClusterRole
│   │   └── developer-role.yaml            # ClusterRole
│   ├── secrets/
│   │   ├── kustomization.yaml
│   │   └── external-secrets.yaml          # ExternalSecret (placeholder for demo)
│   └── observability/
│       ├── kustomization.yaml
│       ├── pod-monitor.yaml               # PodMonitor for AGW metrics
│       └── grafana-dashboard-configmap.yaml
├── overlays/
│   ├── leaf-1/
│   │   ├── kustomization.yaml
│   │   └── patches/
│   │       └── backend-endpoints.yaml     # Cluster-specific secret refs
│   └── leaf-2/
│       ├── kustomization.yaml
│       └── patches/
│           └── backend-endpoints.yaml
├── scripts/
│   ├── install-argocd.sh                  # Bootstrap ArgoCD on hub
│   ├── register-clusters.sh               # Register leaf clusters
│   └── validate.sh                        # End-to-end validation
└── README.md
```

### 3.2 `agw-federated-cluster-1`

Developer self-service for leaf-1 (cluster2).

```
agw-federated-cluster-1/
├── team-enrollment/
│   ├── kustomization.yaml
│   ├── services/
│   │   ├── kustomization.yaml
│   │   ├── enrollment-chatbot.yaml        # SA + Deployment + Service
│   │   ├── data-product-api.yaml          # SA + Deployment + Service
│   │   ├── financial-aid-mcp.yaml         # SA + Deployment + Service
│   │   ├── graph-db-mock.yaml             # SA + Deployment + Service
│   │   └── abac-ext-authz.yaml            # Deployment + Service
│   ├── routes/
│   │   ├── kustomization.yaml
│   │   ├── httproute-openai.yaml          # HTTPRoute to central backend
│   │   ├── httproute-mcp.yaml             # HTTPRoute to central MCP backend
│   │   ├── ingress-routes.yaml            # Chatbot + Grafana hostname routes
│   │   └── reference-grant.yaml           # Cross-namespace backend reference
│   └── policies/
│       ├── kustomization.yaml
│       └── chatbot-rbac.yaml              # ClusterRole + ClusterRoleBinding
├── argocd/
│   └── appproject.yaml                    # AppProject with namespace restrictions
└── README.md
```

### 3.3 `agw-federated-cluster-2`

Same structure as cluster-1. Mirrors the enrollment-agent deployment to demonstrate identical fan-out.

---

## 4. ArgoCD Application Model

### Cluster Generator

All ApplicationSets use the cluster generator with label selectors instead of hardcoded URLs. This decouples git from cluster addressing (critical for local Colima where IPs are dynamic):

```yaml
generators:
  - clusters:
      selector:
        matchLabels:
          agw-role: leaf
```

The install script registers each leaf cluster with ArgoCD and applies the label `agw-role: leaf`. ApplicationSets automatically discover registered clusters.

### ApplicationSets

Six ApplicationSets, one per product group:

| ApplicationSet | Source Type | Generator | Sync Waves |
|---|---|---|---|
| `istio` | Helm (multi-source) | cluster + list (4 components) | 1-4 |
| `agentgateway` | Helm (multi-source) | cluster + list (2 components) | 5-6 |
| `enforcement` | Helm | cluster | 7 |
| `monitoring` | Helm | cluster | 8 |
| `infra-config` | Kustomize | cluster | 10 |
| `cluster-apps` | Kustomize | cluster (with per-cluster repo mapping) | 20 |

### Helm Multi-Source Pattern

Helm ApplicationSets use ArgoCD's multi-source feature to reference values files from the infra repo while pulling charts from upstream registries:

```yaml
spec:
  sources:
    - repoURL: oci://us-docker.pkg.dev/soloio-img/istio-helm
      chart: istiod
      targetRevision: 1.29.0-solo
      helm:
        valueFiles:
          - $values/helm-apps/istio/istiod-values.yaml
          - $values/helm-apps/istio/overlays/{{name}}-istiod-values.yaml
    - repoURL: https://github.com/ably77/agw-federated-infra.git
      targetRevision: main
      ref: values
```

### Sync Wave Summary

```
Wave 1:  Gateway API CRDs + Istio base
Wave 2:  Istio CNI
Wave 3:  Istiod
Wave 4:  ztunnel
Wave 5:  AGW CRDs
Wave 6:  AGW controller
Wave 7:  Kyverno
Wave 8:  Prometheus/Grafana
Wave 10: Infra config (namespaces, backends, policies, mesh, admission)
Wave 20: Developer workloads (services, routes)
```

Gaps between wave numbers leave room for future additions without renumbering.

### AppProjects

**`platform`** -- used by all infra ApplicationSets:
- Source repos: `agw-federated-infra`
- Destinations: all leaf clusters, all namespaces
- Cluster-scoped resources: allowed (ClusterPolicy, ClusterRole, GatewayClass)

**`developers`** -- used by cluster app ApplicationSets:
- Source repos: `agw-federated-cluster-1`, `agw-federated-cluster-2`
- Destinations: respective leaf cluster only, team namespaces only
- Cluster-scoped resources: denied (except ClusterRole/ClusterRoleBinding for demo RBAC)

---

## 5. Resource Decomposition

### Infra Repo (`base/`)

| Source | Infra Path | Resource Types | Rationale |
|---|---|---|---|
| `k8s/namespaces.yaml` | `base/namespaces.yaml` | Namespace | Platform controls ns creation + ambient labels |
| `install.sh` (gateway config) | `base/agw-config/gateway-parameters.yaml` | EnterpriseAgentgatewayParameters | Platform-owned gateway config |
| `install.sh` (gateway) | `base/agw-config/gateway.yaml` | Gateway | Platform-owned listener |
| `install.sh` (access logs) | `base/agw-config/access-logging.yaml` | EnterpriseAgentgatewayPolicy | Centralized observability |
| `install.sh` (tracing) | `base/agw-config/tracing.yaml` | EnterpriseAgentgatewayPolicy | Centralized observability |
| `k8s/gateway/backend.yaml` | `base/backends/openai.yaml` | AgentgatewayBackend | Model registry -- single source of truth |
| `k8s/gateway/mcp-backend.yaml` | `base/backends/mcp-backend.yaml` | AgentgatewayBackend | Model registry |
| `k8s/gateway/guardrails.yaml` | `base/global-policies/guardrails.yaml` | EnterpriseAgentgatewayPolicy | Global safety controls |
| `k8s/gateway/rate-limit.yaml` | `base/global-policies/rate-limit.yaml` | RateLimitConfig + EnterpriseAgentgatewayPolicy | Global token governance |
| `k8s/gateway/ingress.yaml` | `base/ingress/` | Gateway + EnterpriseAgentgatewayParameters | Platform-owned ingress |
| `k8s/mesh/*.yaml` | `base/mesh/*.yaml` | AuthorizationPolicy, Gateway, Telemetry | FERPA boundaries -- must not diverge |
| (new) | `base/admission-policies/*.yaml` | Kyverno ClusterPolicy | Runtime enforcement from ref arch |
| (new) | `base/rbac/*.yaml` | ClusterRole | Scoped access from ref arch |
| (new) | `base/secrets/external-secrets.yaml` | ExternalSecret | Pattern demo (secret created by script) |
| `install.sh` (pod monitor) | `base/observability/pod-monitor.yaml` | PodMonitor | Centralized metrics |
| `k8s/observability/` | `base/observability/grafana-dashboard-configmap.yaml` | ConfigMap | Centralized dashboard |

### Cluster Repos (`team-enrollment/`)

| Source | Cluster Repo Path | Resource Types | Rationale |
|---|---|---|---|
| `k8s/services/*.yaml` | `team-enrollment/services/*.yaml` | SA + Deployment + Service | Developer-owned workloads |
| `k8s/gateway/abac-ext-authz.yaml` | `team-enrollment/services/abac-ext-authz.yaml` | Deployment + Service | Developer-owned workload |
| `k8s/gateway/route.yaml` | `team-enrollment/routes/httproute-openai.yaml` | HTTPRoute | Developer-managed route |
| `k8s/gateway/mcp-backend.yaml` (route) | `team-enrollment/routes/httproute-mcp.yaml` | HTTPRoute | Developer-managed route |
| `k8s/gateway/ingress-routes.yaml` | `team-enrollment/routes/ingress-routes.yaml` | HTTPRoute | Developer-managed route |
| (new) | `team-enrollment/routes/reference-grant.yaml` | ReferenceGrant | Cross-namespace backend ref |
| `k8s/services/enrollment-chatbot.yaml` (RBAC) | `team-enrollment/policies/chatbot-rbac.yaml` | ClusterRole + ClusterRoleBinding | Demo RBAC |

### Key Decisions

- **MCP backend** (`AgentgatewayBackend`) to infra, **HTTPRoute** for it to cluster repo
- **Mesh AuthorizationPolicies** to infra -- FERPA boundaries must not diverge
- **Namespaces** to infra -- ambient mesh label is a platform decision
- **OpenAI secret** not in git -- install script creates it, ExternalSecret in infra shows the pattern

---

## 6. Install Script

### `scripts/install-argocd.sh`

**Prerequisites:**
- 3 Colima clusters running (cluster1, cluster2, cluster3)
- `SOLO_TRIAL_LICENSE_KEY` and `OPENAI_API_KEY` set
- `helm`, `kubectl`, `argocd` CLI available

**Flow:**

1. **Validate** -- check clusters running, env vars set, CLIs available
2. **Install ArgoCD on cluster1** -- `helm install argo/argo-cd` with values, wait for ready
3. **Discover inter-cluster networking** -- for each leaf (cluster2, cluster3), find API server address reachable from cluster1 pods. Strategy: try Colima VM IPs first (`colima ssh <profile> -- hostname -I`), fall back to host gateway + forwarded port
4. **Register leaf clusters** -- create ArgoCD cluster secrets with labels `agw-role: leaf`, `agw-cluster: leaf-1` / `leaf-2`
5. **Apply ArgoCD resources** -- AppProjects, ApplicationSets from the infra repo
6. **Create LLM secrets** -- `kubectl create secret generic enrollment-openai-secret` on cluster2 and cluster3
7. **Wait for sync** -- poll ArgoCD until all Applications Healthy + Synced (timeout 10 min)
8. **Print access info** -- port-forward commands for chatbot, Grafana, ArgoCD UI

### `scripts/register-clusters.sh`

Standalone re-registration for when Colima IPs change after restart.

### `scripts/validate.sh`

Six validation phases:
1. **ArgoCD health** (hub) -- all Applications Synced + Healthy
2. **Infrastructure** (each leaf) -- Istio, AGW, Kyverno, monitoring running
3. **Workloads** (each leaf) -- all deployments ready, waypoint running, mesh enrollment
4. **Enforcement** (each leaf) -- Kyverno blocks rogue backend creation, blocks policy override
5. **Traffic** (each leaf) -- chatbot health, AGW route 200, guardrail blocks PII (422), rate limit headers
6. **GitOps drift** (one leaf) -- manual kubectl change reverted by ArgoCD selfHeal

---

## 7. Overlay Strategy

Overlays patch only what must differ per cluster.

### Istio Overlays

Per-leaf cluster identity for multi-cluster mesh:

```yaml
# helm-apps/istio/overlays/leaf-1-istiod-values.yaml
global:
  multiCluster:
    clusterName: cluster2
  network: cluster2
meshConfig:
  trustDomain: cluster2.local
```

### Infra Config Overlays

Minimal for the reference implementation (identical Colima clusters). Structure demonstrates the pattern:

```yaml
# overlays/leaf-1/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patches:
  - path: patches/backend-endpoints.yaml
```

```yaml
# overlays/leaf-1/patches/backend-endpoints.yaml
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: openai
spec:
  policies:
    auth:
      secretRef:
        name: openai-api-key-leaf-1  # Cluster-specific secret name
```

---

## 8. Kyverno Policies

Three ClusterPolicies from the reference architecture, all enforcing:

### deny-local-backends

Blocks creation of `AgentgatewayBackend` by anyone except the ArgoCD service account. Ensures the model registry is managed exclusively via the infra repo.

### deny-policy-override

Blocks creation of `EnterpriseAgentgatewayPolicy` in the `agentgateway-system` namespace by anyone except ArgoCD. Prevents developers from weakening global guardrails.

### require-guardrail-ref

Blocks creation of `HTTPRoute` resources that don't have a corresponding guardrail policy. Ensures all AI routes are governed.

All three policies exclude the ArgoCD application controller ServiceAccount so that GitOps-managed resources can be applied.

---

## 9. Access Pattern (Colima)

Colima doesn't provide LoadBalancer IPs. Primary access is via port-forward:

```bash
# ArgoCD UI (hub)
kubectl port-forward svc/argocd-server -n argocd 8080:443 --context cluster1

# Enrollment chatbot (leaf-1)
kubectl port-forward svc/enrollment-chatbot -n wgu-demo-frontend 8501:8501 --context cluster2

# Grafana (leaf-1)
kubectl port-forward svc/grafana-prometheus -n monitoring 3000:3000 --context cluster2
```

The install script prints these commands and optionally starts them in the background.
