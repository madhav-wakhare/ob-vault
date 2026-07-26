# Kubernetes Secrets

## What is a Secret?

A Secret is a Kubernetes object used to store sensitive information such as:

- Passwords
- API Keys
- Database Credentials
- TLS Certificates
- SSH Keys
- Tokens

Instead of hardcoding sensitive values inside application code or YAML files, Kubernetes stores them in Secrets.

---

## Why Do We Need Secrets?

### Bad Practice

```yaml
env:
- name: DB_PASSWORD
  value: admin123
```

Problems:

- Password visible in Git repositories.
- Password exposed in manifests.
- Hard to rotate credentials.
- Security risk.

---

### Better Practice

```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password
```

The application receives the password without storing it directly in the Deployment manifest.

---

# How Secrets Are Stored

Example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: YWRtaW4=
  password: YWRtaW4xMjM=
```

---

## Important Note

Secret values are:

```text
Base64 Encoded
```

NOT

```text
Encrypted
```

Example:

```bash
echo "admin123" | base64
```

Output:

```text
YWRtaW4xMjM=
```

Anyone can decode:

```bash
echo "YWRtaW4xMjM=" | base64 -d
```

Output:

```text
admin123
```

---

## Security Warning

Many beginners assume:

```text
Secret = Encryption
```

This is incorrect.

By default:

```text
Secret = Base64 Encoding
```

For actual encryption:

- Enable etcd encryption
- Use HashiCorp Vault
- Use External Secrets Operator (ESO)
- Use cloud secret managers

---

# Secret Architecture

```text
Secret
   ↓
Stored in etcd
   ↓
Referenced by Pod
   ↓
Injected into Container
```

---

# Secret Types

## Opaque

Most common secret type.

Example:

```yaml
type: Opaque
```

Used for:

- Passwords
- API keys
- Tokens

---

## TLS Secret

Stores certificates.

```yaml
type: kubernetes.io/tls
```

Contains:

```text
tls.crt
tls.key
```

Used by:

- Ingress
- HTTPS services

---

## Docker Registry Secret

Stores registry credentials.

```yaml
type: kubernetes.io/dockerconfigjson
```

Used for:

```text
Docker Hub
ECR
GCR
ACR
```

Private image pulls.

---

## Service Account Token Secret

Stores service account credentials.

Usually created automatically by Kubernetes.

---

# Creating Secrets

## Imperative Method

### Generic Secret

```bash
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=admin123
```

Verify:

```bash
kubectl get secrets
```

---

## From File

Example:

```bash
kubectl create secret generic app-config \
  --from-file=config.txt
```

---

# Creating Secret Using YAML

Example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret

type: Opaque

data:
  username: YWRtaW4=
  password: YWRtaW4xMjM=
```

Apply:

```bash
kubectl apply -f secret.yaml
```

---

# StringData vs Data

## data

Requires Base64 encoding.

```yaml
data:
  password: YWRtaW4xMjM=
```

---

## stringData

Plain text values.

```yaml
stringData:
  password: admin123
```

Kubernetes automatically converts it to Base64.

---

## Recommended During Development

```yaml
stringData:
  password: admin123
```

Much easier to read and maintain.

---

# Consuming Secrets

Pods can consume Secrets in two ways:

1. Environment Variables
2. Mounted Files

---

# Method 1: Environment Variables

Secret:

```yaml
stringData:
  password: admin123
```

Deployment:

```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password
```

Application receives:

```text
DB_PASSWORD=admin123
```

---

## Flow

```text
Secret
   ↓
Environment Variable
   ↓
Container
```

---

# Method 2: Volume Mount

Deployment:

```yaml
volumes:
- name: secret-volume
  secret:
    secretName: db-secret

containers:
- name: app
  volumeMounts:
  - name: secret-volume
    mountPath: /etc/secrets
```

Files created:

```text
/etc/secrets/
├── username
└── password
```

Reading:

```bash
cat /etc/secrets/password
```

Output:

```text
admin123
```

---

## Flow

```text
Secret
   ↓
Mounted Volume
   ↓
Files
   ↓
Container
```

---

# Updating Secrets

Update Secret:

```yaml
stringData:
  password: newpassword
```

Apply:

```bash
kubectl apply -f secret.yaml
```

---

## What Happens?

### Environment Variables

```text
Pod Restart Required
```

The container keeps old values until restarted.

---

### Mounted Volumes

```text
Automatically Updated
```

Kubernetes refreshes mounted secret files.

---

# Viewing Secrets

List:

```bash
kubectl get secrets
```

Describe:

```bash
kubectl describe secret db-secret
```

View YAML:

```bash
kubectl get secret db-secret -o yaml
```

Decode value:

```bash
kubectl get secret db-secret \
-o jsonpath='{.data.password}' | base64 -d
```

---

# Namespaces and Secrets

Secrets are namespace-scoped.

Example:

```text
dev namespace
  └── db-secret

prod namespace
  └── db-secret
```

These are different resources.

---

## Pod Restriction

A Pod can only access Secrets from its own namespace.

Example:

```text
Pod in dev
    ↓
Can read:
dev/db-secret

Cannot read:
prod/db-secret
```

---

# Secret Lifecycle

```text
Secret Created
       ↓
Stored in etcd
       ↓
Referenced by Pod
       ↓
Injected as Env Variable or File
       ↓
Application Uses Secret
```

---

# Limitations of Kubernetes Secrets

## Base64 Encoding Only

Not true encryption.

---

## Manual Rotation

Changing passwords requires updates.

---

## No Secret Versioning

Limited secret history.

---

## No Dynamic Credentials

Cannot generate temporary database credentials.

---

## Limited Auditing

Hard to track secret access.

---

# Why Vault Exists

Kubernetes Secrets solve:

```text
Secret Storage
```

Vault solves:

```text
Secret Management
```

Vault provides:

- Encryption
- Dynamic credentials
- Secret rotation
- Auditing
- Access policies
- Secret leasing

---

# Kubernetes Secrets vs Vault

| Kubernetes Secret | Vault |
|-------------------|--------|
| Base64 encoded | Encrypted |
| Stored in etcd | Stored in Vault |
| Manual rotation | Automatic rotation |
| Static secrets | Dynamic secrets |
| Limited auditing | Detailed auditing |
| Built into Kubernetes | External system |

---

# Relationship Between Vault, ESO and Secrets

```text
Vault
   ↓
Stores Secret Securely
   ↓
ESO Fetches Secret
   ↓
Creates Kubernetes Secret
   ↓
Pod Consumes Secret
```

---

# Interview Questions

## What is a Kubernetes Secret?

A Kubernetes object used to store sensitive information such as passwords, tokens, API keys, and certificates.

---

## Are Kubernetes Secrets Encrypted?

No.

By default, they are only Base64 encoded.

Encryption requires additional configuration such as etcd encryption or external secret management solutions.

---

## How can a Pod consume a Secret?

1. Environment Variables
2. Mounted Volumes

---

## Can a Pod access a Secret from another Namespace?

No.

Secrets are namespace-scoped resources.

---

## Why use Vault if Kubernetes already has Secrets?

Vault provides encryption, secret rotation, dynamic credentials, auditing, and centralized secret management, which Kubernetes Secrets do not provide by default.

---

# Key Takeaways

- Secrets store sensitive information.
- Secrets are namespace-scoped.
- Secrets are Base64 encoded, not encrypted.
- Pods consume Secrets through environment variables or mounted volumes.
- Secret updates affect mounted volumes automatically but usually require Pod restarts for environment variables.
- Vault is commonly used when advanced secret management is required.