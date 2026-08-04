The fundamental difference is **who calculates the changes (diff/merge)** before updating the resource.

---

# Client-Side Apply (CSA)

```
Manifest
    │
    ▼
kubectl / Helm (Client)
    │
Calculates the changes
    │
    ▼
API Server
```

- The **client** (`kubectl`, Helm, etc.) compares the YAML with its previously saved state (`last-applied-configuration` annotation).
- It generates a patch and sends that patch to the API server.
- Kubernetes trusts the client.

**Think of it as:**

> "I (the client) have already figured out what needs to change. Please apply it."

---

# Server-Side Apply (SSA)

```
Manifest
    │
    ▼
API Server
    │
Calculates the changes
Tracks field ownership
    │
    ▼
Updates Resource
```

- The client sends the **full desired manifest**.
- The **API server** decides what changed.
- The API server also tracks **who owns each field** (`managedFields`).

**Think of it as:**

> "Here's my desired state. Kubernetes, you decide what needs to change."

---

# Simple analogy

### Client-Side Apply

You edit a Word document yourself and send **only the changed paragraphs** to your friend.

### Server-Side Apply

You send the **entire document** to your friend and say:

> "Compare this with your copy and update only what's different."

Your friend also keeps a record of **who last edited each paragraph**.

---

# Why SSA is better for Argo CD

Suppose:

- Helm manages `replicas`
- Argo CD manages `image`

With **SSA**, Kubernetes knows:

```
replicas  → Helm
image     → Argo CD
```

So it can detect ownership conflicts.

With **CSA**, Kubernetes doesn't have this per-field ownership information.

---

## One-line takeaway

- **Client-Side Apply:** The **client** computes the changes and sends a patch.
- **Server-Side Apply:** The **API server** computes the changes, applies them, and tracks **field ownership** (`managedFields`).