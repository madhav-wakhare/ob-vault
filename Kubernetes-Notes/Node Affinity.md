# Node Affinity

Node Affinity is a rule that tells Kubernetes **which nodes are allowed (or preferred) to run a Pod**.

![[Pasted image 20260712125221.png]]
---

## Hard Requirement (Required Affinity)

```yaml
requiredDuringSchedulingIgnoredDuringExecution:
  nodeSelectorTerms:
  - matchExpressions:
    - key: kubernetes.io/e2e-az-name
      operator: In
      values:
      - e2e-az1
      - e2e-az2
```

### Meaning

> Schedule this Pod only on nodes that have the label:
>
> `kubernetes.io/e2e-az-name=e2e-az1`
>
> OR
>
> `kubernetes.io/e2e-az-name=e2e-az2`

### Example

```text
Node1
  kubernetes.io/e2e-az-name=e2e-az1

Node2
  kubernetes.io/e2e-az-name=e2e-az2

Node3
  kubernetes.io/e2e-az-name=e2e-az3
```

Result:

```text
✅ Node1 -> Allowed
✅ Node2 -> Allowed
❌ Node3 -> Not Allowed
```

If only Node3 exists:

```text
Pod = Pending
```

Because the required condition cannot be satisfied.

---

## Soft Requirement (Preferred Affinity)

```yaml
preferredDuringSchedulingIgnoredDuringExecution:
- weight: 1
  preference:
    matchExpressions:
    - key: another-node-label-key
      operator: In
      values:
      - another-node-label-value
```

### Meaning

> Among the allowed nodes, prefer nodes that also have this additional label.

### Example

```text
Node1
  kubernetes.io/e2e-az-name=e2e-az1
  another-node-label-key=another-node-label-value

Node2
  kubernetes.io/e2e-az-name=e2e-az2
```

Both nodes satisfy the required condition.

Kubernetes will prefer:

```text
✅ Node1
```

because it matches the preferred condition as well.

---

## Simple Analogy

### Job Requirements

**Required**
- Must know Kubernetes

**Preferred**
- Knows Terraform

### Candidates

```text
Alice: Kubernetes + Terraform
Bob: Kubernetes
Charlie: Terraform
```

Result:

```text
✅ Alice (eligible)
✅ Bob (eligible)
❌ Charlie (rejected)
```

Between Alice and Bob, Alice is preferred.

---

## Meaning of IgnoredDuringExecution

```yaml
requiredDuringSchedulingIgnoredDuringExecution
```

### Breakdown

- **DuringScheduling** → Check labels when placing the Pod.
- **IgnoredDuringExecution** → Do not re-check after the Pod is running.

### Example

```text
Pod scheduled on Node1
       ↓
Node label removed
       ↓
Pod continues running
```

The affinity rule is only evaluated during scheduling.

---

## Key Points

- Node Affinity controls which nodes can run a Pod.
- Works using node labels.
- `requiredDuringSchedulingIgnoredDuringExecution`
  - Hard requirement.
  - Pod remains Pending if no matching node exists.
- `preferredDuringSchedulingIgnoredDuringExecution`
  - Soft requirement.
  - Scheduler tries to honor it but may choose another node.
- `IgnoredDuringExecution`
  - Rules are checked only during scheduling.
  - Running Pods are not evicted if labels change later.

---

## Memory Trick

```text
required = MUST run here
preferred = NICE to run here
```