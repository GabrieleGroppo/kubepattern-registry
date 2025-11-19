# `Zombie Frontend Resource`

**ID**: `unreferenced-frontend-resource` | **Type**: `CUSTOM` | **Severity**: `WARNING` | **Category**: `architecture`  
**Docs**: [Krateo Widgets](https://docs.krateo.io/key-concepts/kcp/frontend-widget-api-reference) | **Source**: [GitHub](https://github.com/GabrieleGroppo/kubepattern-registry/blob/krateo-test/definitions/)

---

## Problem

Frontend resources (Tables, Markdown, ...) defined in Krateo Control Plane (KCP) but **never referenced** by structural components (Panel, Row, Page) create architectural overhead and dead configuration.

**Example**: A Table containing critical business data exists in the KCP but isn't included in any Page → wasted processing resources, deployment bloat, and configuration confusion for developers.

---

## Solution

Every frontend widget must be referenced in at least one container's `resourcesRefs.items[]` array to ensure visibility in the UI and maintain a clean, functional Resource Graph.

**Key Concepts**: Resource Graph Consistency, Dead Code Elimination, Architectural Hygiene

---

## Implementation
```yaml
# ✅ CORRECT: Referenced Table
apiVersion: widgets.templates.krateo.io/v1beta1
kind: Table
metadata:
  name: visible-table
  namespace: default
spec:
  widgetData:
    columns:
      - title: Field
        valueKey: field
    pageSize: 5
    data: []

---
apiVersion: widgets.templates.krateo.io/v1beta1
kind: Row
metadata:
  name: my-row
  namespace: default
spec:
  widgetData:
    allowedResources:
      - tables
    items:
      - resourceRefId: visible-table
        size: 24
  resourcesRefs:
    items:
      - id: visible-table              # ✅ Reference establishes visibility
        apiVersion: widgets.templates.krateo.io/v1beta1
        name: visible-table
        namespace: default
        resource: tables
        verb: GET
```

**Anti-pattern** (avoid):
```yaml
# ❌ WRONG: Orphaned Table (Zombie Resource)
apiVersion: widgets.templates.krateo.io/v1beta1
kind: Table
metadata:
  name: zombie-table  # Never referenced by any Panel/Row/Page
  namespace: default
spec:
  widgetData:
    columns:
      - title: Example
        valueKey: example
    pageSize: 5
# This table exists but is invisible to users
```

---

## Detection

**Violates if**: 
- Widget has **zero inbound references** from `Panel`, `Row`, `Column`, or `Page`
- Widget `metadata.name` is not found in any container's `resourcesRefs.items[].name` field
- Total inbound relationship points < 1

**Involves**: `Table`, `Markdown`, `Paragraphs`, `Panel`, `Row`, `Column`, `Page`

---

## Trade-offs

⚠️ **During Development**: May trigger warnings on resources created before UI layout is fully wired (temporarily acceptable during active development)

⚠️ **Template Resources**: If keeping intentionally unreferenced resources as templates for quick rollback, warnings can be suppressed but reduce architectural cleanliness

✅ **Benefits**:
- Cleaner architecture with reduced configuration noise
- Improved maintainability and faster onboarding
- Reduced KCP processing overhead
- Accurate UI ↔ configuration mapping for better debugging

---