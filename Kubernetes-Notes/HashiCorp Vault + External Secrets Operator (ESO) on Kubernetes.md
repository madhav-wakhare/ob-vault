# Production Grade HashiCorp Vault + External Secrets Operator (ESO) on Kubernetes

## Overview

This document describes how to migrate from a **Vault Dev Mode deployment** to a more production-oriented setup using:

- HashiCorp Vault
- Raft Storage Backend
- Kubernetes Authentication
- Vault Policies & Roles
- External Secrets Operator (ESO)
- Kubernetes RBAC
- Persistent Storage
- StatefulSets

Architecture:

```text
┌───────────────────────────────────────────────┐
│ Kubernetes Cluster                            │
│                                               │
│ ┌───────────────────────────────┐             │
│ │ Application Node              │             │
│ │                               │             │
│ │ student-api                   │             │
│ └───────────────────────────────┘             │
│                                               │
│ ┌───────────────────────────────┐             │
│ │ Database Node                 │             │
│ │                               │             │
│ │ PostgreSQL                    │             │
│ └───────────────────────────────┘             │
│                                               │
│ ┌───────────────────────────────┐             │
│ │ Dependent Services Node       │             │
│ │                               │             │
│ │ Vault                         │             │
│ │ External Secrets Operator     │             │
│ └───────────────────────────────┘             │
│                                               │
└───────────────────────────────────────────────┘
```

---

# Why Vault?

Before Vault:

```yaml
apiVersion: v1
kind: Secret
data:
  password: cG9zdGdyZXNfc2VjdXJlX3Bhc3N3b3Jk
```

Anyone can do:

```bash
echo "cG9zdGdyZXNfc2VjdXJlX3Bhc3N3b3Jk" | base64 -d
```

Output:

```text
postgres_secure_password
```

Base64 is:

```text
Encoding
≠
Encryption
```

It only changes representation.

Vault provides:

- Encryption at rest
- Centralized secret management
- Access control
- Audit logging
- Secret rotation
- Dynamic credentials
- Identity-based authentication

---

# High Level Secret Flow

```text
Application
    │
    ▼
Kubernetes Secret
    │
    ▼
External Secrets Operator
    │
    ▼
Vault
```

Vault becomes:

```text
Source of Truth
```

instead of:

```text
Git Repository
```

---

# Phase 1: Replace Dev Mode

## Current State

Most likely running:

```bash
vault server -dev
```

or

```yaml
args:
  - server
  - -dev
```

---

## Why Dev Mode Exists

HashiCorp created Dev Mode for:

- Learning
- Demos
- Local testing

---

## Problems with Dev Mode

```text
❌ Root token exposed
❌ Auto-unseal
❌ No persistence
❌ In-memory storage
❌ No HA
❌ No security
```

If pod restarts:

```text
Everything disappears
```

---

# Phase 2: Configure Persistent Storage

## Problem

Where should Vault store secrets?

Without persistence:

```text
Vault Pod Restart
    ↓
Data Lost
```

---

## Solution

Use PVC.

```text
Vault
  ↓
Persistent Volume Claim
  ↓
Persistent Volume
```

---

## Why?

Secrets survive:

```text
Pod Restart
Node Restart
Container Crash
```

---

# Phase 3: Use Raft Storage

## What is Raft?

Raft is:

```text
Consensus Algorithm
```

Used to keep data synchronized between Vault nodes.

---

## Example

```text
Vault-0
Vault-1
Vault-2
```

Secret written to:

```text
Vault-0
```

Replicated to:

```text
Vault-1
Vault-2
```

---

## Why Use Raft?

Benefits:

```text
No external database needed
Built into Vault
Easy HA setup
Production ready
```

---

## Vault Configuration

```hcl
storage "raft" {
  path = "/vault/data"
}
```

Meaning:

```text
Store encrypted secrets
inside /vault/data
```

---

# Phase 4: Deploy Vault as StatefulSet

## Why Not Deployment?

Deployment:

```text
Pod Names Random

vault-abc
vault-xyz
```

Vault needs stable identity.

---

## StatefulSet

Provides:

```text
vault-0
vault-1
vault-2
```

Stable forever.

---

## Why Important?

Raft members are identified by:

```text
Node Identity
```

Changing names breaks cluster membership.

---

# Phase 5: Initialize Vault

## Command

```bash
vault operator init
```

---

## What Happens Internally?

Vault generates:

```text
Master Encryption Key
```

This key encrypts:

```text
All Secrets
```

stored inside Vault.

---

## Problem

If Vault stored this key directly:

```text
Attacker steals storage
Attacker gets key
All secrets compromised
```

---

## Solution

Split key into multiple pieces.

Example:

```text
Key A
Key B
Key C
Key D
Key E
```

This is called:

```text
Shamir Secret Sharing
```

---

## Output

```text
Unseal Key 1
Unseal Key 2
Unseal Key 3
Unseal Key 4
Unseal Key 5
```

---

# Phase 6: Unseal Vault

## What is Sealed?

Think of Vault as:

```text
Locked Safe
```

Vault has encrypted data.

But doesn't yet have:

```text
Master Key
```

loaded into memory.

---

## State

```text
Vault Storage = Present
Vault Data = Present

Master Key = Missing
```

Therefore:

```text
Vault = Sealed
```

---

## Unseal Process

Provide:

```text
3 of 5 keys
```

Example:

```bash
vault operator unseal
```

3 times.

---

## What Happens Internally?

Vault:

```text
Key 1
+
Key 2
+
Key 3
```

Reconstructs:

```text
Master Encryption Key
```

Loads it into memory.

Now Vault can:

```text
Decrypt secrets
Serve requests
```

---

## After Unseal

```text
Sealed = false
```

Vault becomes operational.

---

# Why Does Vault Seal Again?

After restart:

```text
Memory wiped
```

Master key disappears.

Vault intentionally locks itself.

Reason:

```text
Protection against disk theft
```

---

# Phase 7: Enable Kubernetes Authentication

## Problem

How does Vault know:

```text
ESO is ESO?
```

and not a random pod?

---

## Solution

Use Kubernetes Auth.

Enable:

```bash
vault auth enable kubernetes
```

---

## Result

Vault can trust:

```text
Service Accounts
```

issued by Kubernetes.

---

# Phase 8: Configure Kubernetes Auth Backend

## What Happens?

Vault must validate:

```text
JWT Token
```

coming from Kubernetes.

---

## Flow

```text
ESO Pod
   │
   ▼
JWT Token
   │
   ▼
Vault
   │
   ▼
Kubernetes API
```

Vault asks:

```text
Is this token valid?
```

Kubernetes replies:

```text
Yes
```

or

```text
No
```

---

# Phase 9: Enable KV Secret Engine

## What is a Secret Engine?

Vault plugin responsible for storing secrets.

---

## Enable KV v2

```bash
vault secrets enable -path=kv kv-v2
```

---

## Result

Vault creates:

```text
kv/
```

path.

Example:

```text
kv/dev
```

---

## Store Secret

```bash
vault kv put kv/dev \
database_url="postgresql://..."
```

---

# Phase 10: Create Policies

## What is a Policy?

Vault's version of:

```text
RBAC
```

---

## Example

```hcl
path "kv/data/dev" {
  capabilities = ["read"]
}
```

Meaning:

```text
Can read
Cannot write
Cannot delete
```

---

## Why?

Principle of Least Privilege.

Give only minimum access.

---

# Phase 11: Create Roles

## What is a Role?

Maps:

```text
Kubernetes Identity
        ↓
Vault Policy
```

---

## Example

```text
Service Account
external-secrets
```

gets:

```text
eso-policy
```

---

## Why?

Without role:

```text
Authenticated
≠
Authorized
```

---

# Authentication vs Authorization

Authentication:

```text
Who are you?
```

Authorization:

```text
What can you do?
```

---

# Phase 12: Install External Secrets Operator

## Purpose

ESO acts as:

```text
Secret Synchronization Controller
```

---

## Problem

Applications expect:

```text
Kubernetes Secret
```

not Vault APIs.

---

## ESO Solves

```text
Vault
   ↓
ESO
   ↓
Kubernetes Secret
```

automatically.

---

# Phase 13: Create ClusterSecretStore

## Purpose

Defines:

```text
How ESO talks to Vault
```

---

## Contains

```text
Vault Address
Vault Auth Method
Vault Role
Vault Path
```

---

## Think Of It As

```text
Database Connection Configuration
```

for ESO.

---

# Phase 14: Create ExternalSecret

## Purpose

Defines:

```text
Which secret
should be copied
```

---

## Example

Vault:

```text
kv/dev
```

contains:

```text
database_url
```

---

## ExternalSecret

Maps:

```text
database_url
```

to:

```text
DATABASE_URL
```

inside Kubernetes Secret.

---

# Complete Authentication Flow

```text
ESO Pod Starts
      │
      ▼
Uses Service Account JWT
      │
      ▼
Authenticates to Vault
      │
      ▼
Vault verifies JWT
      │
      ▼
Vault checks Role
      │
      ▼
Role grants Policy
      │
      ▼
Policy allows read access
      │
      ▼
Vault returns secret
      │
      ▼
ESO creates Kubernetes Secret
      │
      ▼
Application consumes secret
```

---

# Kubernetes RBAC

## Purpose

Controls access to Kubernetes resources.

---

## Example

Allow ESO:

```text
Read Secrets
Create Secrets
Update Secrets
```

---

## But Prevent

```text
Delete Namespaces
Delete Nodes
```

---

## Layers of Security

```text
Kubernetes RBAC
        +
Vault Policies
```

Both required.

---

# Future Production Enhancements

## TLS

Problem:

```text
Secrets transmitted as plaintext
```

Solution:

```text
HTTPS
```

---

## Auto Unseal

Current:

```text
Manual unseal
```

Production:

```text
AWS KMS
Azure Key Vault
Google KMS
```

Vault automatically retrieves key.

---

## HA Vault Cluster

```text
vault-0
vault-1
vault-2
```

Benefits:

```text
Node Failure Tolerance
Leader Election
High Availability
```

---

## Network Policies

Allow:

```text
ESO → Vault
```

Deny:

```text
Everything Else
```

---

# Final Architecture

```text
┌───────────────────────────────┐
│ student-api                   │
└──────────────┬────────────────┘
               │
               ▼
┌───────────────────────────────┐
│ Kubernetes Secret             │
└──────────────┬────────────────┘
               │
               ▼
┌───────────────────────────────┐
│ External Secrets Operator     │
└──────────────┬────────────────┘
               │
               ▼
┌───────────────────────────────┐
│ Kubernetes Auth               │
└──────────────┬────────────────┘
               │
               ▼
┌───────────────────────────────┐
│ Vault Role                    │
└──────────────┬────────────────┘
               │
               ▼
┌───────────────────────────────┐
│ Vault Policy                  │
└──────────────┬────────────────┘
               │
               ▼
┌───────────────────────────────┐
│ KV Secret Engine              │
└──────────────┬────────────────┘
               │
               ▼
┌───────────────────────────────┐
│ Raft Storage                  │
└───────────────────────────────┘
```

# Key Interview Takeaway

> Vault is not just a place to store secrets. Vault is an identity-aware secret management system. Kubernetes Secrets merely deliver secrets to workloads, while Vault securely stores, encrypts, audits, rotates, and authorizes access to those secrets. ESO acts as the bridge between Vault and Kubernetes workloads.