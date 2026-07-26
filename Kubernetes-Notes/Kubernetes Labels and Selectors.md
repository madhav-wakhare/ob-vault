# Kubernetes Labels and Selectors

#kubernetes #devops

> [!info] Summary
> Labels are key-value pairs attached to Kubernetes objects. Selectors use those labels to identify and group related objects. Together, they are the primary mechanism Kubernetes uses to connect resources like Deployments, Services, and Pods.

---

## What Are Labels

Labels are arbitrary key-value pairs attached to objects such as Pods, Deployments, and Services. They are used for organization, grouping, and selection.

```yaml
metadata:
  labels:
    app.kubernetes.io/name: nginx
    app.kubernetes.io/instance: my-release
```

Common conventions include:

| Label | Typical Meaning |
|---|---|
| `app.kubernetes.io/name` | The name of the application, often the chart name. |
| `app.kubernetes.io/instance` | A unique identifier for this specific deployment or release. |

---

## What Are Selectors

Selectors are how one object finds another object based on matching labels. This is how a Deployment knows which Pods belong to it, or how a Service knows which Pods to route traffic to.

```yaml
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: nginx
      app.kubernetes.io/instance: my-release
```

> [!warning] Consistency Matters
> The labels defined in `metadata.labels`, `spec.selector.matchLabels`, and `spec.template.metadata.labels` must match exactly. A mismatch means the Deployment cannot correctly find or manage its own Pods.

---

## Why This Gets Repetitive

Because the same labels often need to appear in three or more places within a single manifest, and across multiple manifest files such as `deployment.yaml` and `service.yaml`, this is a common source of copy-paste errors.

Helm addresses this with named templates. See [[Helm Helper Files and Named Templates]] for how to centralize label definitions and reuse them across a chart.

---

## Related Notes
- [[Helm Helper Files and Named Templates]]
- [[Helm Chart Structure]]