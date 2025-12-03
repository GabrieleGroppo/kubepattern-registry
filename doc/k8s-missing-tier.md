# Missing Tier

**ID**: `missing-tier` | **Type**: `FOUNDATIONAL` | **Severity**: `INFO` | **Category**: `governance`
**Docs**: [Kubernetes Recommended Labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/) | **Source**: [GitHub](https://github.com/GabrieleGroppo/kubepattern-registry/tree/main/doc/k8s-missing-tier.json)

-----

## Problem

Kubernetes resources are often deployed without clear architectural categorization labels. Without specific labels identifying the application's layer (e.g., `frontend`, `backend`, `cache`), it becomes difficult to group resources logically for operations such as:

  * Defining **Network Policies** (e.g., "Allow ingress only to the frontend tier").
  * Filtering **Monitoring/Logging** dashboards.
  * Allocating costs or resources based on architectural roles.

**Example**: A cluster contains 50 running Pods. You want to apply a restrictive Network Policy to all database components, but you cannot select them efficiently because they only have generic labels like `app: postgres` or `release: v1`.

-----

## Solution

Enforce a standardized labeling convention that includes a `tier` (or `component`) label on all Pods and Deployments. This label explicitly defines the workload's role within the application architecture.

**Key Concepts**: `Taxonomy`, `Label Selectors`, `Operational Grouping`.

-----

## Implementation

```yaml
# ✅ Correct Example: Tier Label Applied
apiVersion: v1
kind: Pod
metadata:
  name: backend-api
  labels:
    app: payment-service
    # Key implementation line
    tier: backend 
spec:
  containers:
  - name: api
    image: my-api:1.0
```

**Anti-pattern** (avoid):

```yaml
# ❌ Incorrect: Missing architectural context
apiVersion: v1
kind: Pod
metadata:
  name: backend-api
  labels:
    app: payment-service
    # Missing 'tier' label
spec:
  containers: [{ name: api, image: my-api:1.0 }]
```

-----

## Detection

The pattern detects Pods that are part of an application (have an `app` label) but lack the specific classification of which tier they belong to.

**Violates if**:

  * Resource is a `Pod`.
  * `.metadata.labels.app` **exists** (identifying it as a managed application).
  * `.metadata.labels.tier` **does not exist**.

**Involves**: `Pod`, `Deployment`, `StatefulSet`.

-----

## Trade-offs

✅ **Benefits**:

  * **Targeted Security**: Enables precise Network Policies (e.g., `allow-from: tier=frontend` -\> `to: tier=backend`).
  * **Observability**: Allows splitting metrics by tier (e.g., "Frontend latency vs Backend latency").
  * **Automation**: Simplifies CI/CD targeting for specific layers of the stack.
  * **Pattern Recognition**: Make contestual aware patterns easy to spot.

⚠️ **Drawbacks**:

  * **Maintenance**: Requires strict enforcement during code reviews or admission control (like OPA/Kyverno) to ensure labels are not forgotten.

-----