# Kubernetes Operators

## Why Operators Exist

Kubernetes knows how to manage built-in resources such as:

- Pods
- Deployments
- StatefulSets
- Services

Example:

```text
Deployment
    ↓
Deployment Controller
    ↓
Creates and manages Pods
```

The Deployment Controller knows how to keep the desired number of Pods running.

---

## The Problem

Some applications are much more complex than a Deployment.

Example: MySQL

Running MySQL in production involves:

- Creating primary and replica instances
- Configuring replication
- Taking backups
- Restoring backups
- Handling failover
- Upgrading versions safely
- Monitoring cluster health

Kubernetes does not know how to perform these application-specific tasks.

---

## What is an Operator?

An Operator is a Kubernetes controller that contains application-specific operational knowledge.

Example:

```text
MySQL Operator
```

The MySQL Operator knows:

- How to deploy MySQL
- How to scale MySQL
- How to configure replication
- How to perform backups
- How to recover from failures
- How to upgrade safely

Instead of manually performing these tasks, users declare the desired state and the Operator automates the rest.

---

## What is a CRD?

### Problem

By default Kubernetes understands:

```yaml
kind: Pod
kind: Deployment
kind: Service
```

But it does not understand:

```yaml
kind: MySQL
```

---

### Solution

A CRD (Custom Resource Definition) adds a new resource type to Kubernetes.

Example:

```text
Add new resource:
MySQL
```

After installing the CRD:

```yaml
apiVersion: database.example.com/v1
kind: MySQL
metadata:
  name: mysql-prod
spec:
  replicas: 3
```

becomes a valid Kubernetes object.

---

## CRD vs Operator

### CRD

Adds a new resource type.

```text
CRD
 ↓
Kubernetes now understands:
kind: MySQL
```

### Operator

Watches the custom resource and takes actions.

```text
MySQL Resource Created
          ↓
Operator Detects It
          ↓
Creates StatefulSets
Creates PVCs
Creates Services
Configures Replication
Monitors Health
```

---

## How an Operator Works

```text
User Creates MySQL Resource
                ↓
        Kubernetes Stores It
                ↓
        Operator Watches It
                ↓
      Operator Creates Resources
                ↓
    Actual State Matches Desired State
```

---

# Controllers vs Operators

## What is a Controller?

A Controller continuously watches Kubernetes resources and ensures that the actual state matches the desired state.

Example:

```yaml
kind: Deployment
spec:
  replicas: 3
```

Current state:

```text
Desired Pods = 3
Actual Pods  = 2
```

Deployment Controller:

```text
Creates 1 additional Pod
```

Goal:

```text
Desired State = Actual State
```

---

## Examples of Built-in Controllers

- Deployment Controller
- ReplicaSet Controller
- StatefulSet Controller
- DaemonSet Controller
- Job Controller

---

## What is an Operator?

An Operator is a specialized controller with application-specific knowledge.

Example:

```text
MySQL Operator
```

Knows how to:

- Configure replication
- Create backups
- Perform failover
- Upgrade databases
- Restore data

---

## Relationship

```text
Every Operator is a Controller

Not Every Controller is an Operator
```

---

## Controller vs Operator

| Controller | Operator |
|------------|----------|
| Generic automation | Application-specific automation |
| Manages built-in resources | Manages complex applications |
| Usually built into Kubernetes | Usually custom-built |
| Deployment → Pods | MySQL → Database Cluster |
| Limited domain knowledge | Deep application knowledge |

---

## Example Comparison

### Deployment Controller

Input:

```yaml
kind: Deployment
spec:
  replicas: 3
```

Action:

```text
Creates and maintains 3 Pods
```

---

### MySQL Operator

Input:

```yaml
kind: MySQL
spec:
  replicas: 3
```

Action:

```text
Creates StatefulSet
Creates PVCs
Configures Replication
Monitors Health
Creates Backups
Handles Failover
Performs Upgrades
```

---

## Real-Life Analogy

### Controller = Receptionist

Knows general procedures:

```text
Patient Arrives
      ↓
Create Record
      ↓
Assign Room
```

Works for everyone.

---

### Operator = Heart Surgeon

Knows specialized procedures:

```text
Heart Patient Arrives
         ↓
Perform Surgery
         ↓
Monitor Recovery
         ↓
Handle Complications
```

Deep domain expertise.

---

## Another Analogy

```text
CRD = New word added to a dictionary

Operator = Expert who knows what to do with that word
```

Example:

```text
CRD adds:
MySQL

Operator understands:
How to run MySQL
```

---

## Key Takeaways

### CRD

```text
Adds a new Kubernetes resource type.
```

Example:

```text
MySQL
Kafka
Redis
MongoDB
```

---

### Controller

```text
Continuously reconciles desired state with actual state.
```

Example:

```text
Deployment Controller
ReplicaSet Controller
StatefulSet Controller
```

---

### Operator

```text
A specialized controller that uses CRDs and application-specific knowledge to automate the lifecycle of complex applications.
```

Examples:

```text
MySQL Operator
PostgreSQL Operator
Kafka Operator
Prometheus Operator
MongoDB Operator
```

---

## Interview Answer

> A Kubernetes Operator is a specialized controller that uses CRDs to extend Kubernetes and automate deployment, scaling, upgrades, backups, failover, and recovery of complex applications such as databases, Kafka clusters, and monitoring systems.