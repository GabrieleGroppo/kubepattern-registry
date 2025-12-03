# Sidecar

**ID**: `sidecar` | **Type**: `STRUCTURAL` | **Severity**: `INFO` | **Category**: `architecture`
**Docs**: [Kubernetes Patterns](https://www.google.com/search?q=https://kubernetes.io/docs/concepts/workloads/pods/%23uses-of-pods) | **Source**: [GitHub](https://github.com/GabrieleGroppo/kubepattern-registry/tree/main/doc/k8s-sidecar.json)

-----

## Problem

Functionality that supports a main application (such as log shipping, proxying, or configuration updates) is inextricably linked to the application's lifecycle but is deployed as a totally separate Pod.

**Example**: Deploying a `fluentd` logging agent as a standalone Deployment that tries to read logs from a web application running in a different Pod via a shared network volume (NFS/PVC). This introduces network latency, desynchronized lifecycles (the logger might be down while the app is up), and complex volume management.

-----

## Solution

Co-locate the auxiliary container (the "sidecar") in the **same Pod** as the main application container.

By defining multiple containers in a single Pod, they share the same:

1.  **Network Namespace**: They can communicate via `localhost`.
2.  **Lifecycle**: They are scheduled, started, and terminated together.
3.  **Storage**: They can mount and share the same specific volumes (e.g., `emptyDir`) for high-performance local data exchange.

**Key Concepts**: `Multi-container Pods`, `Shared Volumes`, `Lifecycle Coupling`.

-----

## Implementation

```yaml
# ✅ Correct Example: Sidecar Pattern
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
spec:
  volumes:
  - name: shared-logs
    emptyDir: {}
  containers:
  # 1. Main Application
  - name: main-app
    image: my-app:1.0
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/app
  # 2. Sidecar Container (Log Shipper)
  - name: log-sidecar
    image: fluentd:latest
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/app # Reads from the same local disk location
```

**Anti-pattern** (avoid):
Separating tightly coupled processes into distinct Pods.

```yaml
# ❌ Incorrect: Main App Pod
apiVersion: v1
kind: Pod
metadata: { name: main-app }
spec:
  containers: [{ name: app, image: my-app }] 
  # Missing the sidecar that provides essential support
---
# ❌ Incorrect: Separated "Sidecar" running alone
apiVersion: v1
kind: Pod
metadata: { name: logger-standalone }
spec:
  containers: [{ name: fluentd, image: fluentd }]
```

-----

## Detection

The provided pattern logic identifies a "Distributed Monolith" or "Split Sidecar" anti-pattern where components that *should* be sidecars are detected as separate entities.

**Violates if**:

  * A Pod identified as a "Main App" (labels: `frontend`, `backend`, `api`) exists.
  * A separate Pod identified as a "Sidecar Candidate" (labels/names: `fluentd`, `logging`, `filebeat`) exists.
  * These two separate Pods appear to rely on a shared volume relationship (e.g., both mounting the same PVC to exchange data), indicating they should be merged.

**Involves**: `Pod`, `Volume`, `Deployment`.

-----

## Trade-offs

⚠️ **Drawbacks**

  * **Coupled Scaling**: The sidecar scales 1:1 with the main container. If the main app needs 10 replicas, you run 10 sidecars, which might be resource-inefficient compared to a DaemonSet (one agent per node).
  * **Single Point of Failure**: If the sidecar container crashes and the Pod restart policy is `Always`, Kubernetes might restart the entire environment depending on the probe configuration.

✅ **Benefits**

  * **Atomic Scheduling**: Guaranteed that the helper is always running where the app is running.
  * **Low Latency**: Communication over `localhost` or local volumes is significantly faster than service-to-service networking.
  * **Security**: No need to expose the helper service to the wider cluster network; it can listen only on localhost.