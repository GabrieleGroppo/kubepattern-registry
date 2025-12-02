# `Limit Range Predictable Demands`

**ID**: `limit-range-predictable-demands` | **Type**: `FOUNDATIONAL` | **Severity**: `WARNING` | **Category**: `architecture`  
**Docs**: [Kubernetes LimitRange](https://kubernetes.io/docs/concepts/policy/limit-range/) | **Source**: [GitHub](https://github.com/GabrieleGroppo/kubepattern-registry/tree/main/doc/limit-range-predictable-demands.json)

---

## Problem

Namespaces without **LimitRange** policies allow unrestricted resource consumption per Pod/Container, creating risk of resource exhaustion and unpredictable scheduling behavior. A single misconfigured Pod can request excessive resources, blocking legitimate workloads from scheduling or consuming disproportionate node capacity.

**Example**: A developer accidentally deploys a Pod requesting 128GB memory in a namespace without limits. The Pod blocks scheduling of 50+ smaller workloads until the error is discovered, causing service disruptions.

---

## Solution

Define **LimitRange** in every application namespace to enforce default resource constraints and maximum/minimum boundaries for Pods and Containers. This provides a safety net when developers forget to specify resource requests/limits and prevents catastrophic resource requests.

**Key Concepts**: Namespace Resource Governance, Default Resource Injection, Resource Boundary Enforcement

---

## Implementation
```yaml
# ✅ CORRECT: Namespace with LimitRange
apiVersion: v1
kind: Namespace
metadata:
  name: production-app

---
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production-app
spec:
  limits:
  - max:                    # Maximum per container
      cpu: "2"
      memory: "4Gi"
    min:                    # Minimum per container
      cpu: "100m"
      memory: "128Mi"
    default:                # Applied if not specified
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:         # Applied if not specified
      cpu: "250m"
      memory: "256Mi"
    type: Container
  - max:                    # Maximum per Pod
      cpu: "4"
      memory: "8Gi"
    type: Pod
```

**Anti-pattern** (avoid):
```yaml
# ❌ WRONG: Namespace without LimitRange
apiVersion: v1
kind: Namespace
metadata:
  name: unprotected-namespace
# Missing LimitRange = unbounded resource requests allowed
# Pods can request arbitrary amounts of CPU/memory
```

---

## Detection

**Violates if**: 
- Namespace exists with no `LimitRange` resources in the same namespace
- Missing `IS_NAMESPACE_OF` relationship between Namespace and LimitRange
- Namespace is not in system namespace exclusion list (`kube-system`, `kube-public`, `kube-node-lease`, `krateo-system`)

**Involves**: `Namespace`, `LimitRange`

---

## Trade-offs

⚠️ **Generic Defaults**: LimitRange defaults may not fit all workload types in a namespace (microservices vs batch jobs). Requires thoughtful default selection or per-workload overrides

⚠️ **Retroactive Application**: Adding LimitRange to existing namespaces doesn't affect already-running Pods (only new/restarted Pods)

✅ **Benefits**:
- Prevents resource exhaustion from misconfigured Pods
- Reduces scheduler thrashing from impossible resource requests
- Enforces consistent baseline resource management
- Protects against accidental resource monopolization
- Simplifies developer experience with sensible defaults

---
