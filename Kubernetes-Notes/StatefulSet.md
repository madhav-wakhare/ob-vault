# StatefulSet Networking & Headless Services

## Why This Topic Matters

Many engineers initially assume:

```text
StatefulSet
      ↓
Provides Stable IP Addresses
```

This is incorrect.

A StatefulSet provides:

- Stable Pod Names
- Stable DNS Names
- Stable Storage
- Ordered Deployment & Scaling

It does **NOT** provide stable Pod IPs.

StatefulSet
    ↓
volumeClaimTemplates
    ↓
PVC created
(vault-data-vault-0)
    ↓
StorageClass
    ↓
PV created
(pvc-af078...)
    ↓
PVC Bound
    ↓
Pod starts

---

# The Core Concept

## StatefulSet Guarantees Identity, Not IP

Consider a StatefulSet:

```text
mysql-0
mysql-1
mysql-2
```

Initial state:

```text
mysql-0 → 10.244.0.5
mysql-1 → 10.244.0.6
mysql-2 → 10.244.0.7
```

---

Suppose:

```text
mysql-1 crashes
```

Kubernetes recreates it:

```text
mysql-1 → 10.244.2.10
```

Notice:

```text
Pod Name = Same
Pod IP   = Changed
```

---

## Important Rule

```text
StatefulSet guarantees:

✓ Stable Name

✗ Stable IP
```

---

# The Real Problem

Imagine a MySQL replication cluster.

```text
Master
  ↓
Replica-1
  ↓
Replica-2
```

---

If replicas communicate using Pod IPs:

```text
10.244.0.5
10.244.0.6
10.244.0.7
```

and a Pod restarts:

```text
10.244.0.6
      ↓
10.244.2.10
```

Replication breaks.

---

Applications need:

```text
A Stable Way To Find Pods
```

instead of:

```text
Temporary Pod IPs
```

---

# Solution: Headless Service

## What is a Headless Service?

A Headless Service is a Service without a ClusterIP.

Example:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: mysql

spec:
  clusterIP: None

  selector:
    app: mysql

  ports:
  - port: 3306
```

---

## What Does `clusterIP: None` Mean?

Normal Service:

```text
Create ClusterIP
Perform Load Balancing
```

Headless Service:

```text
No ClusterIP
No Load Balancing
Expose Individual Pod DNS Records
```

---

# Normal Service Networking

Suppose:

```text
nginx-1
nginx-2
nginx-3
```

Service:

```text
nginx-service
```

---

Traffic Flow:

```text
Client
   ↓
Service
   ↓
Random Pod
```

---

Visual:

```text
Client
   ↓

ClusterIP Service

   ↓
 ┌───────┐
 │ LB    │
 └───────┘

 ↓    ↓    ↓

Pod1 Pod2 Pod3
```

---

The client does NOT know which Pod receives traffic.

---

# Headless Service Networking

StatefulSet:

```text
mysql-0
mysql-1
mysql-2
```

Headless Service:

```text
mysql
```

---

Instead of creating:

```text
One ClusterIP
```

Kubernetes creates:

```text
DNS Records For Each Pod
```

---

# DNS Records Created

Kubernetes automatically creates:

```text
mysql-0.mysql.default.svc.cluster.local

mysql-1.mysql.default.svc.cluster.local

mysql-2.mysql.default.svc.cluster.local
```

---

# DNS Resolution

Suppose:

```text
mysql-0
IP: 10.244.0.5
```

DNS:

```text
mysql-0.mysql
```

resolves to:

```text
10.244.0.5
```

---

Suppose:

```text
mysql-1
IP: 10.244.0.6
```

DNS:

```text
mysql-1.mysql
```

resolves to:

```text
10.244.0.6
```

---

# Pod Restart Scenario

Initial state:

```text
mysql-1
IP: 10.244.0.6
```

DNS:

```text
mysql-1.mysql
      ↓
10.244.0.6
```

---

Pod crashes.

Kubernetes recreates:

```text
mysql-1
IP: 10.244.2.10
```

---

DNS automatically updates:

```text
mysql-1.mysql
      ↓
10.244.2.10
```

Applications continue using:

```text
mysql-1.mysql
```

without knowing the IP changed.

---

# How StatefulSet and Headless Service Work Together

StatefulSet:

```text
Provides Identity
```

Headless Service:

```text
Provides DNS Mapping
```

---

Together:

```text
mysql-0
     ↓
mysql-0.mysql
     ↓
Current IP
```

---

# Complete Flow

## Step 1

Create Headless Service

```yaml
clusterIP: None
```

---

## Step 2

Create StatefulSet

```text
mysql-0
mysql-1
mysql-2
```

---

## Step 3

Pods Receive IPs

```text
mysql-0 → 10.244.0.5
mysql-1 → 10.244.0.6
mysql-2 → 10.244.0.7
```

---

## Step 4

Headless Service Creates DNS

```text
mysql-0.mysql
mysql-1.mysql
mysql-2.mysql
```

---

## Step 5

Applications Use DNS

```text
Replica
   ↓
mysql-0.mysql
   ↓
DNS Lookup
   ↓
Current IP
```

---

# Why Not Use a Normal Service?

Normal Service:

```text
mysql-service
```

returns:

```text
One ClusterIP
```

Example:

```text
10.96.0.10
```

---

Traffic:

```text
Client
   ↓
10.96.0.10
   ↓
Random MySQL Pod
```

---

This is bad for:

```text
MySQL
Kafka
ZooKeeper
MongoDB
Elasticsearch
```

because they need:

```text
Specific Node Identities
```

---

# Real-World Example: Kafka

Kafka Brokers:

```text
broker-0
broker-1
broker-2
```

Each broker has:

```text
Unique Identity
```

---

Broker-1 must know:

```text
Broker-0
Broker-2
```

specifically.

Not:

```text
Any Random Broker
```

---

Therefore Kafka uses:

```text
broker-0.kafka

broker-1.kafka

broker-2.kafka
```

provided by a Headless Service.

---

# Real-World Example: MongoDB Replica Set

Members:

```text
mongo-0
mongo-1
mongo-2
```

Replication configuration:

```text
mongo-0.mongo
mongo-1.mongo
mongo-2.mongo
```

---

If a Pod restarts:

```text
IP changes
```

but:

```text
DNS name remains same
```

Replication continues.

---

# StatefulSet Networking Architecture

```text
                Headless Service
                  (No ClusterIP)

                         │
                         │
                         ▼

            ┌─────────────────────────┐
            │ DNS Records Created     │
            └─────────────────────────┘

                         │

     ┌───────────────────┼───────────────────┐

     ▼                   ▼                   ▼

mysql-0.mysql     mysql-1.mysql     mysql-2.mysql

     ▼                   ▼                   ▼

10.244.0.5        10.244.0.6        10.244.0.7
```

---

# StatefulSet + PVC + Headless Service

The complete StatefulSet architecture:

```text
mysql-0
   │
   ├── DNS
   │      ↓
   │  mysql-0.mysql
   │
   └── PVC
          ↓
       Disk-0


mysql-1
   │
   ├── DNS
   │      ↓
   │  mysql-1.mysql
   │
   └── PVC
          ↓
       Disk-1


mysql-2
   │
   ├── DNS
   │      ↓
   │  mysql-2.mysql
   │
   └── PVC
          ↓
       Disk-2
```

---

# Interview Questions

## Do StatefulSet Pods Get IP Addresses?

Yes.

Every StatefulSet Pod gets a normal Pod IP.

Example:

```text
mysql-0 → 10.244.0.5
mysql-1 → 10.244.0.6
```

---

## Are StatefulSet Pod IPs Stable?

No.

If a Pod is recreated:

```text
IP may change
```

---

## What Remains Stable?

```text
Pod Name

DNS Name

Persistent Storage
```

---

## Why Is Headless Service Required?

To provide:

```text
Stable DNS Records
```

for each StatefulSet Pod.

---

## Does Headless Service Load Balance Traffic?

No.

```text
clusterIP: None
```

means:

```text
No ClusterIP

No Load Balancing

Direct Pod Access
```

---

# Memory Tricks

## Deployment

```text
Hotel Rooms

Room Number Doesn't Matter
```

---

## StatefulSet

```text
Bank Lockers

Identity Matters
```

---

## Headless Service

```text
Phone Directory
```

Maps:

```text
Person Name
      ↓
Current Address
```

---

## StatefulSet Networking

```text
Pod IP
      ↓
Temporary House Address

DNS Name
      ↓
Permanent Contact Name
```

---

# Final Mental Model

```text
StatefulSet
      ↓
Stable Identity

Headless Service
      ↓
Stable DNS

DNS
      ↓
Current Pod IP

Pod
      ↓
Persistent Storage (PVC)
```

StatefulSets do **not provide stable IPs**.

They provide **stable identities**, and Headless Services ensure those identities always resolve to the Pod's current IP.