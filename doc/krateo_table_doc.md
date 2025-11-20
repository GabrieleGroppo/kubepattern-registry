# `Table Not Referenced`

**ID**: `krateo-table-not-referenced` | **Type**: `CUSTOM` | **Severity**: `WARNING` | **Category**: `architecture`  
**Docs**: [Krateo Widgets](https://docs.krateo.io/key-concepts/kcp/frontend-widget-api-reference) | **Source**: [GitHub](https://github.com/GabrieleGroppo/kubepattern-registry/tree/main/doc/table-not-referenced-pattern.json)

---

## Problem

Tables in Krateo Control Plane that are **not referenced** by any widget (Panel, Row, Column, Page) represent orphaned resources consuming cluster capacity without providing value. These zombie resources create configuration clutter, increase maintenance overhead, and make the architecture harder to understand.

**Example**: A developer creates a Table to display deployment metrics but forgets to wire it into the dashboard Page. The Table resource remains in the cluster, consuming memory and processing cycles, but users never see it.

---

## Solution

Every Table must be referenced in at least one container widget's `resourcesRefs.items[]` array to establish visibility in the UI. This ensures the Resource Graph remains clean and all configured resources serve an active purpose.

**Key Concepts**: Resource Graph Integrity, Dead Configuration Elimination, Widget Lifecycle Management

---

## Implementation
```yaml
# ✅ CORRECT: Referenced Table
apiVersion: widgets.templates.krateo.io/v1beta1
kind: Table
metadata:
  name: deployment-metrics
  namespace: production
spec:
  widgetData:
    columns:
      - title: Deployment
        valueKey: name
      - title: Replicas
        valueKey: replicas
    pageSize: 10

---
apiVersion: widgets.templates.krateo.io/v1beta1
kind: Panel
metadata:
  name: metrics-dashboard
  namespace: production
spec:
  widgetData:
    items:
      - resourceRefId: deployment-metrics
        size: 24
  resourcesRefs:
    items:
      - id: deployment-metrics      # ✅ Establishes reference
        apiVersion: widgets.templates.krateo.io/v1beta1
        name: deployment-metrics
        namespace: production
        resource: tables
        verb: GET
```

**Anti-pattern** (avoid):
```yaml
# ❌ WRONG: Orphaned Table
apiVersion: widgets.templates.krateo.io/v1beta1
kind: Table
metadata:
  name: unused-metrics
  namespace: production
spec:
  widgetData:
    columns:
      - title: Field
        valueKey: field
# No Panel, Row, Column, or Page references this Table
# Resource exists but provides zero value
```

---

## Detection

**Violates if**: 
- Table has **zero inbound REFERENCES_KRATEO relationships** from Panel, Row, Column, or Page
- `minRelationshipPoints: 1` is not satisfied
- Table exists in non-system namespace but is never referenced

**Involves**: `Table`, `Panel`, `Row`, `Column`, `Page`

---

## Trade-offs

⚠️ **Development Phase**: During active development, tables may be temporarily unreferenced before UI wiring is complete (acceptable as transient state)

⚠️ **Template Libraries**: Organizations maintaining table templates for reuse may keep intentionally unreferenced resources (can suppress warnings but reduces clarity)

✅ **Benefits**:
- Reduced cluster resource consumption
- Cleaner configuration with no dead code
- Faster debugging and maintenance
- Accurate documentation of active UI components

---