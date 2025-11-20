# `Health Probe Missing`

**ID**: `health-probe-missing` | **Type**: `FOUNDATIONAL` | **Severity**: `WARNING` | **Category**: `Reliability`  
**Docs**: [Kubernetes Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) | **Source**: [GitHub](https://github.com/GabrieleGroppo/kubepattern-registry/tree/main/doc/health-probe-missing.json)

---

## Problem

Pods without **liveness and readiness probes** leave Kubernetes blind to application health. The kubelet cannot detect hung processes, deadlocked threads, or failed startups, resulting in traffic routed to non-functional containers and indefinite waiting for crashed applications to self-recover.

**Example**: A web service enters a deadlock state (process alive but not responding to requests). Without probes, Kubernetes continues routing traffic to the broken Pod for hours until manual intervention, causing user-facing errors and SLA breaches.

---

## Solution

Configure both **livenessProbe** (detects and restarts unhealthy containers) and **readinessProbe** (prevents traffic to containers not ready to serve requests) for every user-facing container. This enables Kubernetes to automatically maintain service health and route traffic intelligently.

**Key Concepts**: Self-Healing, Traffic Management, Container Health Monitoring, Startup Dependencies

**Probe Types**:
- **Liveness**: Restarts container if check fails (detects deadlocks, hung processes)
- **Readiness**: Removes Pod from Service endpoints if check fails (prevents premature traffic)
- **Startup**: Delays liveness checks for slow-starting apps (optional, prevents premature kills)

---

## Implementation
```yaml
# ✅ CORRECT: Pod with health probes
apiVersion: v1
kind: Pod
metadata:
  name: healthy-app
  namespace: production
spec:
  containers:
  - name: web-server
    image: nginx:1.21
    ports:
    - containerPort: 80
    
    livenessProbe:          # Restart if unhealthy
      httpGet:
        path: /health
        port: 80
      initialDelaySeconds: 15
      periodSeconds: 10
      timeoutSeconds: 3
      failureThreshold: 3
    
    readinessProbe:         # Remove from Service if not ready
      httpGet:
        path: /ready
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
      timeoutSeconds: 2
      failureThreshold: 2
```

**Anti-pattern** (avoid):
```yaml
# ❌ WRONG: No health probes
spec:
  containers:
  - name: web-server
    image: nginx:1.21
    ports:
    - containerPort: 80
    # Missing livenessProbe and readinessProbe
    # Kubernetes cannot detect application failures
```

---

## Detection

**Violates if**: 
- `spec.containers[*].livenessProbe` does NOT exist OR
- `spec.containers[*].readinessProbe` does NOT exist
- Pod is not in system namespace exclusion list

**Involves**: `Pod`, `Deployment`, `StatefulSet`, `DaemonSet`

---

## Trade-offs

⚠️ **Probe Configuration**: Incorrect timeouts or thresholds can cause false positives (premature restarts) or false negatives (slow detection). Requires tuning based on application startup time and response patterns

⚠️ **Batch/Job Workloads**: Short-lived Jobs or batch processors may not benefit from probes (acceptable to skip)

✅ **Benefits**:
- Automatic recovery from application deadlocks and crashes
- Zero-downtime deployments with traffic-aware rollouts
- Faster detection of service degradation
- Improved MTTR (Mean Time To Recovery)
- Protection against serving traffic during startup

---