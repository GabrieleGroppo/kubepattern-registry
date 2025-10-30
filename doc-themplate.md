
-----

# `[Pattern Name]`

## ℹ️ Intro

  * **Version**: `kubepattern.it/v1`
  * **id**: `[pattern-id-ex: resource-limit]`
  * **Name**: `[Readable pattern name, ex: Resource Limit Predictable Demands]`
  * **Type**: `[Type, ex: FOUNDATIONAL, BEHAVIORAL, STRUCTURAL]`
  * **Severity**: `[Violation severity, ex: CRITICAL, WARNING, INFO]`
  * **Category**: `[Impact area, ex: architecture, security, reliability, cost-optimization]`
  * **docUrl**: `https://kubernetes.io/docs/home/`
  * **gitUrl**: `https://learn.microsoft.com/it-it/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-design`

-----

## ❗ Problem (Why)

[Clearly and concisely describe the problem this pattern solves. What is the **anti-pattern** (the wrong way of doing things)? What are the **concrete risks** (e.g., instability, unexpected costs, security vulnerabilities) if this pattern is not applied?

Use specific examples, like "memory leak," "noisy neighbor," or "privilege escalation," to make the risk tangible. The goal of this section is to answer the question: "Why should I care about this?"]

-----

## ✅ Solution (How)

[Describe the technical and conceptual solution. Explain the key components of the solution (e.g., what `requests` and `limits` are, or what a `NetworkPolicy` is).

Explain *why* this solution mitigates the problem described earlier. If relevant, explain related concepts that the solution enables or influences (e.g., QoS Classes, principle of "least privilege," etc.).]

-----

## 🛠️ Implementation Example

[Provide one or more minimal but complete YAML code blocks demonstrating the **correct** implementation of the pattern. This serves as a "copy-paste" reference for the user.

Be sure to comment the key lines that implement the pattern.]

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-correct-pattern
spec:
  containers:
  - name: my-container
    image: nginx
    # Example comment: these lines implement the pattern
    resources:
      requests:
        memory: "256Mi"
        cpu: "500m"
      limits:
        memory: "256Mi"
        cpu: "500m"
```

-----

## 🔗 Involved Resources

[List the Kubernetes API objects relevant to this pattern and briefly describe their specific role.]

  * **`Kind`**: [Brief description of its role in the pattern (e.g., `Pod`: the resource where the pattern is applied).]
  * **`Kind`**: [Brief description (e.g., `LimitRange`: a policy that can enforce this pattern at the Namespace level).]
  * **`Kind`**: [Brief description (e.g., `Deployment`: the controller that manages the Pods to which the pattern is applied).]

-----

## 📊 Violation Metrics (How to find it)

[Explain how to programmatically identify resources that *violate* this pattern (i.e., that implement the anti-pattern). This is crucial for scanning tools.]

### Fields to Match

[List the specific YAML fields, using `path.to.field` notation, that must be inspected to detect the violation.]

The pattern is violated if, for any container, one or more of the following fields are **not** defined (are `null` or absent):

  * `spec.containers[].resources.limits.cpu`
  * `spec.containers[].resources.limits.memory`

### Detection Logic (Optional)

[If the logic is more complex than a simple null check (e.g., requires comparing multiple fields), describe it here.]

  * Example: "Violation occurs IF `spec.containers[].securityContext.privileged` is `true`."
  * Example: "Violation occurs IF `spec.containers[].resources.requests.cpu` \> `spec.containers[].resources.limits.cpu`."

-----

## ✨ Benefits and Considerations

[Summarize the advantages and any "trade-offs" to consider before blindly applying the pattern.]

### Benefits

  * [Benefit 1, e.g: Improved cluster stability and predictability.]
  * [Benefit 2, e.g: Cost reduction through more efficient bin-packing.]
  * [Benefit 3, e.g: Increased security and workload isolation.]

### Considerations (Trade-offs)

  * [Consideration 1, e.g: Requires "rightsizing" analysis of applications; setting limits too low can cause OOMKilled.]
  * [Consideration 2, e.g: Can be complex to apply to legacy workloads or those with an unknown consumption profile.]
