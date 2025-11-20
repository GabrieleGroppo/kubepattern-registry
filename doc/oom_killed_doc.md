# `OOM Killed Resource Suggestion`

**ID**: `oom-killed-resource-suggestion` | **Type**: `BEHAVIORAL` | **Severity**: `CRITICAL` | **Category**: `Performance`  
**Docs**: [Kubernetes Memory Management](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/) | **Source**: [GitHub](https://github.com/GabrieleGroppo/kubepattern-registry/tree/main/doc/oom-killed-resource-suggestion.json)

---

## Problem

Containers terminated with **OOMKilled** (Out Of Memory) indicate memory limits are insufficient for actual workload demands. This causes immediate service disruption through forced container termination, data loss for in-flight transactions, and degraded user experience from repeated restart cycles with exponential backoff.

**Example**: A Python data processing service with 1GB memory limit loads a 1.5GB dataset into memory, triggering instant OOMKill. The container restarts, re-attempts processing, and immediately crashes again in a CrashLoopBackOff pattern, halting all data pipeline operations.

---

## Solution

Analyze memory usage patterns and increase memory limits to accommodate peak workload demands plus safety margin:
1. **Profile memory consumption**: Use monitoring tools (Prometheus, cAdvisor) to identify peak usage
2. **Increase limits**: Set memory limits to `peak_usage * 1.2` (20% safety margin)
3. **Optimize application**: Consider memory optimization if usage is excessive (caching strategies, streaming vs loading)
4. **Monitor post-change**: Verify OOMKills cease after limit increase

**Key Concepts**: Memory Pressure, Container Limits Enforcement, Working Set Size, Peak vs Average Usage

---

## Implementation
```yaml
# ✅ CORRECT: Sufficient memory limits
apiVersion: v1
kind: Pod
metadata:
  name: data-processor
  namespace: production
spec:
  containers:
  - name: python-worker
    image: python:3.9
    resources:
      requests:
        memory: "1Gi"
        cpu: "500m"
      limits:
        memory: "2Gi"        # ✅ Accommodates 1.5GB dataset + overhead
        cpu: "1000m"
    
    # Add memory monitoring sidecar (optional)
    env:
    - name: PYTHONMALLOC
      value: "malloc"        # Better memory tracking
```

**Anti-pattern** (avoid):
```yaml
# ❌ WRONG: Insufficient memory causing OOMKilled
spec:
  containers:
  - name: python-worker
    image: python:3.9
    resources:
      limits:
        memory: "512Mi"    # ❌ Too small for actual workload
        cpu: "500m"
    # When processing 1.5GB dataset:
    # 1. Container memory usage exceeds 512Mi
    # 2. Kernel OOM killer terminates container
    # 3. lastState.terminated.reason = "OOMKilled"
    # 4. Pod enters CrashLoopBackOff
```

---

## Detection

**Violates if**: 
- `status.containerStatuses[*].lastState.terminated.reason` equals `"OOMKilled"`
- Pod is not in system namespace exclusion list
- Container was terminated by kernel OOM killer due to exceeding memory limits

**Involves**: `Pod`, `Deployment`, `StatefulSet`, `DaemonSet`, `LimitRange`, `ResourceQuota`

---

## Trade-offs

⚠️ **Memory Leaks**: Simply increasing limits may mask application memory leaks. Investigate unexpected growth patterns before blindly increasing limits

⚠️ **Cost vs Reliability**: Higher limits increase cluster costs. Balance between resource efficiency and stability based on workload criticality

✅ **Benefits**:
- Immediate identification of under-provisioned workloads
- Prevents service disruption from memory exhaustion
- Clear action signal (increase limits) vs ambiguous failures
- Enables data-driven capacity planning
- Reduces cascading failures from OOMKilled Pods

---