# `Resource Quotas Predictable Demands`

**ID**: `resource-quotas-predictable-demands` | **Type**: `FOUNDATIONAL` | **Severity**: `WARNING` | **Category**: `architecture`  
**Docs**: [Kubernetes ResourceQuota](https://kubernetes.io/docs/concepts/policy/resource-quotas/) | **Source**: [GitHub](https://github.com/GabrieleGroppo/kubepattern-registry/tree/main/doc/resource-quotas-predictable-demands.json)

---

## Problem

Namespaces without **ResourceQuota** policies allow unbounded aggregate resource consumption, enabling a single namespace to monopolize cluster capacity and starve other tenants. This creates noisy neighbor scenarios, unpredictable cost allocation, and inability to enforce organizational resource budgets.

**Example**: A development team deploys 100 Pods in their namespace without quotas, consuming 80% of cluster CPU. Production workloads in other namespaces experience scheduling failures and degraded performance due to insufficient remaining capacity, even though their namespace should have guaranteed resources.

---

## Solution

Define **ResourceQuota** for every application namespace to cap total resource consumption and ensure fair cluster resource distribution. Quotas prevent runaway resource usage from misconfigured HorizontalPodAutoscalers, bulk deployments, or resource-intensive workloads.

**Key Concepts**: Multi-tenancy, Capacity Planning, Fair Resource Allocation, Cost Control, Namespace Isolation

**Common Quota Types**:
- **Compute**: CPU/memory requests and limits aggregated across namespace
- **Object counts**: Max Pods, Services, ConfigMaps, Secrets
- **Storage**: PersistentVolumeClaim requests
- **Extended resources**: GPUs, custom resources

---

## Implementation
```yaml
# ✅ CORRECT: Namespace with ResourceQuota
apiVersion: v1
kind: Namespace
metadata:
  name: team-alpha

---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: team-alpha
spec:
  hard:
    requests.cpu: "20"           # Max 20 CPU cores requested
    requests.memory: "40Gi"      # Max 40GB memory requested
    limits.cpu: "40"             # Max 40 CPU cores limit
    limits.memory: "80Gi"        # Max 80GB memory limit
    
    pods: "50"                   # Max 50 Pods
    services: "20"               # Max 20 Services
    persistentvolumeclaims: "10" # Max 10 PVCs
    
    # Optional: Object count limits
    configmaps: "50"
    secrets: "50"
```

**Anti-pattern** (avoid):
```yaml
# ❌ WRONG: Namespace without ResourceQuota
apiVersion: v1
kind: Namespace
metadata:
  name: unbounded-namespace
# Missing ResourceQuota
# Namespace can consume unlimited cluster resources
# Risks: resource exhaustion, cost overruns, unfair allocation
```

---

## Detection

**Violates if**: 
- Namespace exists with no `ResourceQuota` resources in the same namespace
- Missing `IS_NAMESPACE_OF` relationship between Namespace and ResourceQuota
- Namespace is not in system namespace exclusion list (`kube-system`, `kube-public`, `kube-node-lease`, `krateo-system`)

**Involves**: `Namespace`, `ResourceQuota`, `LimitRange`

---

## Trade-offs

⚠️ **Quota Sizing**: Setting quotas too low blocks legitimate workload growth; too high defeats the purpose. Requires monitoring actual usage patterns and adjusting quotas based on team needs

⚠️ **Operational Overhead**: Teams hitting quota limits experience deployment failures requiring manual quota increases, adding operational friction

✅ **Benefits**:
- Prevents single namespace from monopolizing cluster resources
- Enables cost allocation and chargeback per namespace/team
- Protects cluster stability from runaway resource consumption
- Enforces organizational resource governance policies
- Simplifies capacity planning with predictable resource boundaries

---
