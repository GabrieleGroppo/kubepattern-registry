# `High Restart Count Resource Suggestion`

**ID**: `high-restart-count-resource-suggestion` | **Type**: `BEHAVIORAL` | **Severity**: `WARNING` | **Category**: `Performance`  
**Docs**: [Kubernetes Pod Troubleshooting](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/) | **Source**: [GitHub](https://github.com/GabrieleGroppo/kubepattern-registry/tree/main/doc/high-restart-count-resource-suggestion.json)

---

## Problem

Pods with **high restart counts** (>3) signal instability caused by resource constraints (OOMKilled, CPU throttling), application bugs, or environmental issues. Frequent restarts degrade service reliability through repeated downtime windows, waste node resources on failed container initializations, and mask underlying resource sizing problems.

**Example**: A Node.js service with 512MB memory limit experiences occasional memory spikes under load, causing OOMKills every few hours. Over a week, the Pod accumulates 50+ restarts, creating intermittent 503 errors and degraded user experience that goes unnoticed until customers complain.

---

## Solution

Investigate restart root cause and resolve the instability:
1. **Memory pressure**: Check for OOMKilled in `lastState.terminated.reason` → increase memory limits
2. **CPU throttling**: Monitor CPU usage patterns → adjust CPU limits or optimize code
3. **Failed health probes**: Review probe configuration for false positives → tune thresholds
4. **Application bugs**: Analyze container logs for crashes → fix code or dependencies

**Key Concepts**: Container Lifecycle Events, Resource Monitoring, Stability Metrics, Graceful Degradation

---

## Implementation
```yaml
# ✅ CORRECT: Properly resourced Pod with monitoring
apiVersion: v1
kind: Pod
metadata:
  name: stable-service
  namespace: production
spec:
  containers:
  - name: nodejs-app
    image: node:16
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
      limits:
        memory: "1Gi"        # ✅ Headroom above typical usage
        cpu: "1000m"         # ✅ Prevents CPU throttling
    
    livenessProbe:
      httpGet:
        path: /health
        port: 3000
      initialDelaySeconds: 30
      periodSeconds: 10
      failureThreshold: 3    # ✅ Tolerates transient failures
      timeoutSeconds: 5
    
    readinessProbe:
      httpGet:
        path: /ready
        port: 3000
      periodSeconds: 5
      failureThreshold: 2
```

**Anti-pattern** (avoid):
```yaml
# ❌ WRONG: Under-resourced Pod causing frequent restarts
spec:
  containers:
  - name: nodejs-app
    image: node:16
    resources:
      limits:
        memory: "256Mi"    # ❌ Too small - causes OOMKills
        cpu: "100m"        # ❌ Causes startup delays and throttling
    
    livenessProbe:
      httpGet:
        path: /health
        port: 3000
      initialDelaySeconds: 5  # ❌ Too short for slow startup
      periodSeconds: 5
      failureThreshold: 1     # ❌ No tolerance for transient failures
    # Result: restart count quickly exceeds threshold
```

---

## Detection

**Violates if**: 
- `status.containerStatuses[*].restartCount` is greater than `3`
- Pod is not in system namespace exclusion list
- Pattern indicates chronic instability rather than one-time incident

**Involves**: `Pod`, `Deployment`, `StatefulSet`, `DaemonSet`, `ResourceQuota`

---

## Trade-offs

⚠️ **Threshold Tuning**: The "3 restarts" threshold may be too sensitive for some workloads (batch jobs with expected failures) or too lenient for critical services. Adjust pattern threshold based on use case

⚠️ **Time Window**: Pattern doesn't distinguish between "3 restarts in 5 minutes" (critical) vs "3 restarts over 30 days" (normal). Consider time-based analysis for better signal

✅ **Benefits**:
- Early detection of resource under-provisioning
- Prevents service degradation from becoming customer-visible
- Guides capacity planning and right-sizing efforts
- Reduces operational toil from repeated restart investigations

---