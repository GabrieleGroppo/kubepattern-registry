# `Pod Priority Predictable Demands`

**ID**: `pod-priority-predictable-demands` | **Type**: `FOUNDATIONAL` | **Severity**: `WARNING` | **Category**: `architecture`  
**Docs**: [Kubernetes Pod Priority](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/) | **Source**: [GitHub](https://github.com/GabrieleGroppo/kubepattern-registry/tree/main/doc/pod-priority-predictable-demands.json)

---

## Problem

Pods without explicit **priority** (priority=0 or undefined) are treated as lowest priority during resource contention, making them vulnerable to eviction when nodes experience resource pressure. This creates unpredictable behavior where critical services may be preempted to make room for less important workloads scheduled earlier.

**Example**: A critical API service Pod with no priority runs on a node that reaches memory capacity. A batch analytics job with higher priority needs scheduling, causing Kubernetes to evict the API Pod, disrupting user-facing traffic to accommodate the less critical batch workload.

---

## Solution

Assign **PriorityClass** to every Pod based on business criticality to control scheduling and eviction behavior. Define cluster-wide priority tiers (e.g., critical=1000, high=750, normal=500, low=250) and ensure critical services receive protection from preemption.

**Key Concepts**: Pod Preemption, Priority-based Scheduling, Quality of Service, Resource Contention Resolution

**Priority Tiers**:
- **System-critical** (1000-2000): Core infrastructure (DNS, monitoring)
- **Business-critical** (750-999): User-facing services, APIs
- **Normal** (500-749): Standard applications
- **Low-priority** (0-499): Batch jobs, non-critical workloads

---

## Implementation
```yaml
# ✅ CORRECT: Define PriorityClass
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: business-critical
value: 900                  # Higher value = higher priority
globalDefault: false
description: "Critical user-facing services"

---
apiVersion: v1
kind: Pod
metadata:
  name: api-service
  namespace: production
spec:
  priorityClassName: business-critical  # ✅ Explicit priority
  containers:
  - name: api
    image: myapp:v1
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
      limits:
        memory: "1Gi"
        cpu: "1000m"
```

**Anti-pattern** (avoid):
```yaml
# ❌ WRONG: No priority defined
apiVersion: v1
kind: Pod
metadata:
  name: critical-api
  namespace: production
spec:
  # Missing priorityClassName
  # Defaults to priority=0 (lowest)
  containers:
  - name: api
    image: myapp:v1
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
  # Pod vulnerable to eviction during resource pressure
```

---

## Detection

**Violates if**: 
- `spec.priorityClassName` is empty/null OR
- `spec.priority` is empty/null OR
- `spec.priority` equals `0`
- Pod is not in system namespace exclusion list

**Involves**: `Pod`, `Deployment`, `StatefulSet`, `DaemonSet`, `PriorityClass`

---

## Trade-offs

⚠️ **Preemption Risk**: High-priority Pods can evict lower-priority workloads, potentially disrupting batch jobs or non-critical services. Requires careful priority design

⚠️ **Starvation**: Excessive high-priority Pods can starve lower-priority workloads indefinitely. Monitor scheduling success rates across priority tiers

✅ **Benefits**:
- Protects critical services during resource contention
- Predictable eviction behavior under pressure
- Better resource allocation aligned with business priorities
- Improved stability for user-facing workloads
- Clear operational hierarchy for incident response

---