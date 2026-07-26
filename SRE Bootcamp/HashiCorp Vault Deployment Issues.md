# HashiCorp Vault Deployment on Minikube - Troubleshooting & Root Cause Analysis

## Environment

- Kubernetes: Minikube
- Namespace: `vault-ns`
- Workload Type: `StatefulSet`
- Storage Backend: File Storage (`/vault/data`)
- Service: Headless Service
- Node Scheduling:

```yaml
nodeSelector:
  type: dependent_services
```

---

# Objective

Deploy a single-node HashiCorp Vault instance on the Kubernetes node labeled:

```yaml
type: dependent_services
```

using:

- StatefulSet
- Persistent Storage
- ConfigMap-based configuration
- Service Account
- Headless Service

---

# Issues Encountered During Deployment

## Issue 1: Vault Listener Port Conflict

### Symptoms

```text
Error initializing listener of type tcp:
listen tcp4 0.0.0.0:8200: bind: address already in use
```

Initially, it appeared that another process was already using port `8200`.

---

### Investigation

Checked processes on the target Minikube node:

```bash
minikube ssh -n minikube-m03
ss -tunlp
```

Result:

```text
No process listening on port 8200
```

This confirmed the issue was not caused by:

- Kubernetes Service
- Node Networking
- Host-Level Process

The conflict existed inside the pod/container namespace.

---

## Additional Investigation: Was Vault Actually Starting Twice?

### Initial Assumption

Because Vault reported:

```text
listen tcp4 0.0.0.0:8200: bind: address already in use
```

it appeared that Vault might be starting twice inside the container.

However, this was never directly verified.

We did not inspect running processes inside the container using:

```bash
ps aux
```

Therefore:

> "Vault was definitely starting twice"

cannot be stated as a confirmed fact.

---

### What We Know For Certain

Vault attempted to bind:

```text
0.0.0.0:8200
```

and received:

```text
bind: address already in use
```

Since no node-level process was using the port, the conflict existed within the container namespace.

---

### Why a Double-Start Was Suspected

Initially, the deployment relied on:

```yaml
args:
  - server
  - -config=/vault/config/vault.hcl
```

without explicitly defining a command.

The Vault image contains its own:

```dockerfile
ENTRYPOINT
```

Kubernetes combines:

```text
ENTRYPOINT + args
```

Resulting in something conceptually similar to:

```bash
docker-entrypoint.sh server -config=/vault/config/vault.hcl
```

If the entrypoint script itself starts Vault and the provided arguments trigger another startup path, a listener conflict can occur.

This led to the hypothesis that Vault was being started more than once.

---

### What Actually Fixed It

The deployment was updated to:

```yaml
command:
  - vault

args:
  - server
  - -config=/vault/config/vault.hcl
```

Kubernetes now executes:

```bash
vault server -config=/vault/config/vault.hcl
```

directly.

After making this change:

- Vault started successfully.
- Listener conflicts disappeared.
- Pod became healthy.

---

### Final Conclusion on the Port Conflict

#### Confirmed Facts

✅ Explicitly defining:

```yaml
command:
  - vault
```

resolved the issue.

✅ The conflict was inside the container namespace.

✅ No node-level process was using port `8200`.

#### Most Likely Explanation

The most likely root cause was an interaction between:

- Vault image startup behavior (`ENTRYPOINT`)
- Supplied container arguments (`args`)

which resulted in a listener conflict on port `8200`.

#### Important Note

The statement:

> "Vault was starting twice"

remains a hypothesis and was never directly proven.

The confirmed statement is:

> Explicitly defining the container command removed the listener conflict and allowed Vault to start successfully.

---

# Issue 2: Service Port and api_addr Mismatch

## Earlier Configuration

Vault Listener:

```hcl
listener "tcp" {
  address = "0.0.0.0:8200"
}
```

Service:

```yaml
ports:
  - port: 8203
    targetPort: 8200
```

Vault API Address:

```hcl
api_addr = "http://vault.vault-ns.svc.cluster.local:8200"
```

---

## Why This Was Problematic

Vault advertises itself using:

```hcl
api_addr
```

If clients connect through a different port than the advertised address:

- UI redirects may fail.
- CLI requests may fail.
- Internal communication can break.
- HA configurations can encounter issues.

---

## Resolution

Aligned everything to port `8200`.

Service:

```yaml
ports:
  - port: 8200
    targetPort: 8200
```

Vault Config:

```hcl
api_addr = "http://vault.vault-ns.svc.cluster.local:8200"
```

---

# Issue 3: Invalid Vault Configuration

## Symptoms

```text
unknown or unsupported field vault found in configuration
```

---

## Cause

A previous version of the configuration contained an unsupported block:

```hcl
vault {
 ...
}
```

Vault does not support a top-level:

```hcl
vault {}
```

block.

---

## Resolution

Used a valid Vault configuration:

```hcl
ui = true

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = 1
}

storage "file" {
  path = "/vault/data"
}

api_addr = "http://vault.vault-ns.svc.cluster.local:8200"
```

---

# Issue 4: IPC_LOCK Warning

## Symptoms

```text
Couldn't start vault with IPC_LOCK.
Disabling IPC_LOCK.
```

---

## Cause

Vault attempts to use Linux `mlock()` to prevent sensitive data from being swapped to disk.

Containers require:

```yaml
securityContext:
  capabilities:
    add:
      - IPC_LOCK
```

for this capability.

---

## Resolution

Added:

```yaml
securityContext:
  capabilities:
    add:
      - IPC_LOCK
```

---

## Important Note

This warning was **not the reason Vault failed**.

Vault can still start successfully without IPC_LOCK in a local Minikube environment.

---

# Issue 5: ConfigMap Read-Only Warnings

## Symptoms

```text
chown: /vault/config/... Read-only file system
```

---

## Cause

Vault's startup script attempts:

```bash
chown /vault/config
```

However:

- ConfigMaps are mounted as read-only volumes.
- Kubernetes prevents ownership changes on ConfigMap mounts.

---

## Resolution

No action required.

These warnings are harmless and do not prevent Vault startup.

---

# Why StatefulSet Was Chosen

Vault stores data locally:

```hcl
storage "file" {
  path = "/vault/data"
}
```

StatefulSet provides:

- Stable Pod Identity
- Stable Storage
- Persistent PVC Association

Using:

```yaml
volumeClaimTemplates:
```

automatically creates:

```text
vault-storage-vault-0
```

for:

```text
vault-0
```

Benefits:

- Pod restarts do not lose data.
- PVC remains associated with pod identity.
- Suitable for stateful workloads.

---

# volumeClaimTemplates Explained

`volumeClaimTemplates` are only supported by StatefulSets.

Example:

```yaml
volumeClaimTemplates:
  - metadata:
      name: vault-storage
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
```

Kubernetes automatically creates:

```text
vault-storage-vault-0
```

for:

```text
vault-0
```

This is one of the major reasons StatefulSets are preferred for:

- Vault
- PostgreSQL
- MySQL
- Kafka
- Elasticsearch

---

# Final Working Configuration Highlights

## Vault Config

```hcl
ui = true

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = 1
}

storage "file" {
  path = "/vault/data"
}

api_addr = "http://vault.vault-ns.svc.cluster.local:8200"
```

---

## Service

```yaml
spec:
  clusterIP: None
  selector:
    app: vault
  ports:
    - port: 8200
      targetPort: 8200
```

---

## StatefulSet Container

```yaml
containers:
  - name: vault
    image: hashicorp/vault:2.0

    securityContext:
      capabilities:
        add:
          - IPC_LOCK

    command:
      - vault

    args:
      - server
      - -config=/vault/config/vault.hcl
```

---

## Persistent Storage

```yaml
volumeClaimTemplates:
  - metadata:
      name: vault-storage
    spec:
      storageClassName: standard
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
```

---

# Final Architecture

```text
Namespace: vault-ns
        │
        ▼
StatefulSet (vault)
        │
        ├── Pod: vault-0
        │      ├── Vault Server
        │      ├── IPC_LOCK Enabled
        │      ├── ConfigMap → /vault/config
        │      └── PVC → /vault/data
        │
        ▼
Headless Service (vault)
        │
        ▼
vault.vault-ns.svc.cluster.local:8200
```

---

# Key Learnings

## 1. Port Conflicts

`bind: address already in use`

does not always indicate a node-level port conflict.

Always verify:

```bash
ss -tunlp
```

before assuming networking issues.

---

## 2. Service and api_addr Consistency

Keep these aligned:

```text
Listener Port = Service Port = api_addr Port
```

Example:

```text
8200 = 8200 = 8200
```

---

## 3. ConfigMap Warnings

```text
Read-only file system
```

errors on ConfigMap mounts are usually harmless.

---

## 4. IPC_LOCK

Recommended for security.

Not mandatory for local Minikube deployments.

---

## 5. Stateful Workloads

Use StatefulSets when:

- Stable identity is required.
- Persistent storage is required.
- Each replica needs its own volume.

---

# Final Outcome

✅ Vault successfully deployed.

✅ Pod scheduled on node:

```yaml
type: dependent_services
```

✅ Persistent storage provisioned using PVC.

✅ Configuration loaded from ConfigMap.

✅ Vault API reachable through:

```text
vault.vault-ns.svc.cluster.local:8200
```

✅ StatefulSet and PVC functioning correctly.

✅ Vault running successfully in namespace:

```text
vault-ns
```