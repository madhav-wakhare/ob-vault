### Finalizer in Argocd:

A finalizer in Argo CD (`resources-finalizer.argoproj.io`) is ==a Kubernetes metadata tag that tells the Argo CD controller to perform a cascading delete of all managed child resources (like Deployments and Services) before allowing the main Application custom resource itself to be fully deleted==.

When you delete an Argo CD Application with a finalizer, the **cascading delete** ensures that all the real-world parts (like your application's web servers, databases, and network settings) are safely destroyed together. Nothing gets left behind to clutter your cluster.


# Resource Tracking Problem (Helm → Argo CD)

## Background

Both **Helm** and **Argo CD** need a way to identify which Kubernetes resources belong to them.

This is called **resource tracking**.

---

# Helm Resource Tracking

Helm adds labels such as:

```yaml
metadata:
  labels:
    app.kubernetes.io/instance: eso-config
    app.kubernetes.io/managed-by: Helm
```

It also adds ownership annotations:

```yaml
metadata:
  annotations:
    meta.helm.sh/release-name: eso-config
    meta.helm.sh/release-namespace: eso-ns
```

---

# Argo CD Default Resource Tracking

By default, Argo CD also tracks resources using the label:

```yaml
app.kubernetes.io/instance
```

---

# The Problem

If both Helm and Argo CD manage the same label:

```text
Helm
   │
   ▼
app.kubernetes.io/instance
   ▲
   │
Argo CD
```

Both controllers may attempt to update the same metadata field.

This can lead to:

- Resource ownership confusion
- Unnecessary OutOfSync status
- Metadata conflicts
- Difficult Helm → Argo CD migrations

> **Note:** In a Server-Side Apply (SSA) environment (e.g., Helm 4), ownership is tracked per field by the Kubernetes API server, making these conflicts more significant.

---

# Production Solution

Configure Argo CD to track resources using **annotations** instead of labels.

```yaml
configs:
  cm:
    application.resourceTrackingMethod: annotation
```

Argo CD will then add annotations such as:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/tracking-id: ...
```

instead of modifying:

```yaml
metadata:
  labels:
    app.kubernetes.io/instance: ...
```

---

# Result

```text
Before

Helm ───────────────► app.kubernetes.io/instance
Argo CD ───────────► app.kubernetes.io/instance

❌ Both use the same label.

----------------------------------------

After

Helm ───────────────► Labels

Argo CD ───────────► Annotations

✅ No tracking conflict.
```

---

# Key Takeaway

During a **Helm → Argo CD** migration, changing Argo CD's resource tracking method to **annotations** avoids competing over the same tracking label and makes adoption of existing Helm-managed resources safer.


# Server-side Diff (Argo CD)

## What is it?

Instead of comparing the **Git manifest** directly with the **live resource**, Argo CD asks the Kubernetes API server:

> "If I applied this manifest, what would the resource look like?"

The API server performs a **server-side dry-run** and returns the predicted object for comparison.

---

## Why use it?

Kubernetes automatically adds or modifies fields such as:

- `resourceVersion`
- `managedFields`
- `uid`
- `creationTimestamp`
- Default values

A simple YAML comparison can incorrectly report these as drift.

Server-side diff compares the **predicted live object** with the **actual live object**, significantly reducing false `OutOfSync` reports.

---

## Enable it

```yaml
configs:
  params:
    controller.diff.server.side: true
```

---

## Comparison

### Without Server-side Diff

```text
Git Manifest
      │
      ▼
Direct YAML Comparison
      │
      ▼
Live Resource

❌ May report false differences
```

### With Server-side Diff

```text
Git Manifest
      │
      ▼
API Server (Dry Run Apply)
      │
      ▼
Predicted Resource
      │
      ▼
Compare with Live Resource

✅ More accurate diff
```

---

## When is it most useful?

- Helm 4 (Server-Side Apply)
- Argo CD managing existing resources
- Clusters with multiple controllers (ESO, cert-manager, Vault, Istio, etc.)
- Brownfield GitOps migrations

---

> **Key Takeaway:** Server-side diff uses the Kubernetes API server to calculate the expected resource before comparing it with the live object, producing more accurate diffs and reducing false `OutOfSync` states.


# `crds.install: true`

## What does it do?

Tells the Argo CD Helm chart to **install its Custom Resource Definitions (CRDs)** during installation.

```yaml
crds:
  install: true
```

Without the CRDs, Kubernetes does not recognize Argo CD resources such as:

- `Application`
- `ApplicationSet`
- `AppProject`

Creating one of these resources would fail with an error like:

```text
no matches for kind "Application" in version "argoproj.io/v1alpha1"
```

---

## Why is it needed?

CRDs extend the Kubernetes API by introducing new resource types.

Installing the CRDs first allows Kubernetes to understand Argo CD's custom resources before they are created.

---

## Example

```text
Install Argo CD CRDs
        │
        ▼
Kubernetes learns about:
- Application
- ApplicationSet
- AppProject
        │
        ▼
Application YAML can now be created successfully.
```

---

## Key Takeaway

`crds.install: true` ensures Argo CD's custom resource definitions are installed so Kubernetes can recognize and manage resources like `Application`, `ApplicationSet`, and `AppProject`.