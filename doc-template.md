# `[Pattern Name]`

**ID**: `pattern-id` | **Type**: `FOUNDATIONAL` | **Severity**: `CRITICAL` | **Category**: `reliability`  
**Docs**: [Kubernetes](https://kubernetes.io/docs/) | **Source**: [GitHub](https://github.com/...)

---

## Problem

[2-3 sentences: anti-pattern + concrete risk with example]

**Example**: [Specific problem scenario]

---

## Solution

[Technical solution description in 3-4 sentences. Focus on WHAT to do and WHY it works]

**Key Concepts**: [List key concepts if needed: e.g., QoS Classes, Least Privilege]

---

## Implementation
```yaml
# Minimal commented example (10-20 lines max)
apiVersion: v1
kind: Pod
metadata:
  name: correct-example
spec:
  containers:
  - name: app
    image: nginx
    # Key implementation lines
    resources:
      requests:
        memory: "256Mi"
      limits:
        memory: "256Mi"
```

**Anti-pattern** (avoid):
```yaml
# Incorrect example in 3-5 lines
spec:
  containers:
  - name: app
    resources: {} # ❌ Missing limits
```

---

## Detection

**Violates if**: 
- `spec.containers[].resources.limits.memory` is `null` OR
- `spec.containers[].resources.limits.cpu` is `null`

**Involves**: `Pod`, `Deployment`, `LimitRange`

---

## Trade-offs

⚠️ **[Only if relevant]**
- [Trade-off 1 with concrete impact]
- [Trade-off 2 if applicable]

✅ **Benefits**: [Concise list of 2-3 bullets]

---
