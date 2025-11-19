# `Resource Limit Predictable Demands`

**ID**: `resource-limit-predictable-demands` | **Type**: `FOUNDATIONAL` | **Severity**: `WARNING` | **Category**: `architecture`  
**Docs**: [Kubernetes Resources](https://kubernetes.io/docs/concepts/workloads/pods/resource-limit-predictable-demands/) | **Source**: [GitHub](https://github.com/GabrieleGroppo/kubepattern-registry/tree/main/doc/resource-limit-pattern.md)

---

## Problem

Omitting **resource limits** (CPU and memory) on Kubernetes Pods introduces serious stability and predictability risks. Without constraints, a Pod can **consume resources uncontrollably**: it may saturate node memory triggering OOMKill or eviction of other workloads (memory leak scenario), or monopolize CPU causing performance degradation and latency for co-located applications (noisy neighbor effect).

**Example**: A Pod with a memory leak gradually consumes all available memory on a node, causing Kubernetes to evict critical workloads. Without limits, the scheduler cannot properly bin-pack Pods, leading to inefficient resource utilization and unpredictable autoscaling behavior.

---

## Solution

Set **`requests`** and **`limits`** for both CPU and memory on every container to ensure predictable resource consumption and proper QoS classification.

- **`requests`**: Minimum guaranteed resources used by the scheduler for node placement decisions
- **`limits`**: Maximum resource ceiling; exceeding memory limits triggers OOMKill, exceeding CPU limits causes throttling

This configuration enables Kubernetes to manage workloads effectively and protect co-located applications from resource starvation.

**Key Concepts**: QoS Classes (Guaranteed, Burstable, BestEffort), Scheduler Bin-Packing, Resource Isolation

**QoS Classes**:
- **Guaranteed**: `requests` == `limits` (best for predictable workloads, highest priority)
- **Burstable**: `requests` < `limits` (can burst if resources available)
- **BestEffort**: No `requests`/`limits` (first to be evicted under pressure)

---

## Implementation

```yaml
# ✅ CORRECT: Guaranteed QoS (ideal for predictable demands)
apiVersion: v1
kind: Pod
metadata:
  name: predictable-pod
spec:
  containers:
  - name: my-container
    image: nginx
    resources:
      requests:
        memory: "256Mi"
        cpu: "500m"      # 0.5 CPU core
      limits:
        memory: "256Mi"  # Same as requests = Guaranteed QoS
        cpu: "500m"
```

**Anti-pattern** (avoid):
```yaml
# ❌ WRONG: No limits (BestEffort QoS)
spec:
  containers:
  - name: my-container
    image: nginx
    resources: {}  # Missing requests and limits
    # Pod can consume unlimited resources
```

---

## Detection

**Violates if**: 
- `spec.containers[].resources.limits.cpu` is `null` OR
- `spec.containers[].resources.limits.memory` is `null`

**Involves**: `Pod`, `Deployment`, `StatefulSet`, `DaemonSet`, `LimitRange`, `ResourceQuota`

---

## Trade-offs

⚠️ **Rightsizing Required**: Setting limits too low causes OOMKills; too high wastes cluster capacity. Requires profiling application resource consumption patterns

⚠️ **Legacy Workloads**: Difficult to apply to applications with unknown or highly variable resource profiles without extensive testing

✅ **Benefits**:
- Improved cluster stability and predictability
- Better bin-packing efficiency and cost optimization
- Protection against noisy neighbor scenarios
- Reliable autoscaling behavior
- Higher QoS priority during resource pressure

---