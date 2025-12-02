# `CrashLoopBackOff Resource Suggestion`

**ID**: `crash-loop-backoff-resource-suggestion` | **Type**: `BEHAVIORAL` | **Severity**: `CRITICAL` | **Category**: `Performance`  
**Docs**: [Kubernetes Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/) | **Source**: [GitHub](https://github.com/GabrieleGroppo/kubepattern-registry/tree/main/doc/crash-loop-backoff-resource-suggestion.json)

---

## Problem

Pods in **CrashLoopBackOff** state indicate repeated container failures, often caused by insufficient resources (OOMKilled, CPU starvation during startup), missing dependencies, or application bugs. This state creates service downtime, wastes node resources on failed restarts, and delays incident detection through exponential backoff periods.

**Example**: A Java application configured with 256MB memory limit crashes during startup because the JVM heap requires 512MB minimum. Kubernetes repeatedly restarts the Pod with increasing delays (10s, 20s, 40s...), but each attempt fails instantly, leaving the service unavailable for extended periods.

---

## Solution

Investigate CrashLoopBackOff root cause immediately and address the underlying issue:
1. **Resource constraints**: Increase memory/CPU limits if logs show OOMKilled or startup timeouts
2. **Startup failures**: Add startup probes with longer `initialDelaySeconds` for slow-starting apps
3. **Configuration errors**: Validate ConfigMaps, Secrets, and environment variables
4. **Dependency issues**: Ensure external services are reachable and healthy

**Key Concepts**: Container Exit Codes, Exponential Backoff, Startup Time Optimization, Resource Right-Sizing

---

## Implementation
```yaml
# ✅ CORRECT: Sufficient resources and startup time
apiVersion: v1
kind: Pod
metadata:
  name: stable-app
  namespace: production
spec:
  containers:
  - name: java-app
    image: openjdk:11-jre
    resources:
      requests:
        memory: "512Mi"      # Sufficient for JVM startup
        cpu: "500m"
      limits:
        memory: "1Gi"        # Headroom for peak usage
        cpu: "1000m"
    
    startupProbe:            # Allow slow startup
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
      failureThreshold: 10   # 100s total startup time
    
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      periodSeconds: 10
```

**Anti-pattern** (avoid):
```yaml
# ❌ WRONG: Insufficient resources causing CrashLoopBackOff
spec:
  containers:
  - name: java-app
    image: openjdk:11-jre
    resources:
      limits:
        memory: "128Mi"    # ❌ Too small for JVM
        cpu: "100m"        # ❌ Insufficient for startup
    # Missing startup probe = premature liveness kills
    # Container crashes repeatedly: CrashLoopBackOff
```

---

## Detection

**Violates if**: 
- `status.containerStatuses[*].state.waiting.reason` equals `"CrashLoopBackOff"`
- Pod is not in system namespace exclusion list
- Container exhibits repeated restart pattern with exponential backoff

**Involves**: `Pod`, `Deployment`, `StatefulSet`, `DaemonSet`, `ResourceQuota`, `LimitRange`

---

## Trade-offs

⚠️ **Root Cause Complexity**: CrashLoopBackOff has multiple potential causes (resources, config, code bugs, network). Requires log analysis and debugging to identify correct fix

⚠️ **False Positives**: Transient startup failures (race conditions, external service delays) may trigger pattern temporarily during deployment

✅ **Benefits**:
- Immediate visibility into critical service failures
- Prevents wasted resources on doomed restart attempts
- Guides operators to resource-related issues quickly
- Reduces MTTR by surfacing problem early

---
