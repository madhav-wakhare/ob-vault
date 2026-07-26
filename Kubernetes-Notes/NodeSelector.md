# Node Selector

Node Selector is the simplest way to control which node a Pod can run on.

It works by matching Pod requirements with node labels.

---

## Example

### Nodes

```text
Node1
  kubernetes.io/e2e-az-name=e2e-az1

Node2
  kubernetes.io/e2e-az-name=e2e-az2

Node3
  kubernetes.io/e2e-az-name=e2e-az3
```

### Pod Definition

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  nodeSelector:
    kubernetes.io/e2e-az-name: e2e-az1

  containers:
  - name: nginx
    image: nginx
```

### Meaning

> Schedule this Pod only on nodes having:

```text
kubernetes.io/e2e-az-name=e2e-az1
```

### Result

```text
✅ Node1 -> Allowed
❌ Node2 -> Not Allowed
❌ Node3 -> Not Allowed
```

If Node1 is unavailable:

```text
Pod = Pending
```

because no matching node exists.

---

## Multiple Labels Example

### Nodes

```text
Node1
  env=prod
  disk=ssd

Node2
  env=prod
  disk=hdd

Node3
  env=dev
  disk=ssd
```

### Pod Definition

```yaml
spec:
  nodeSelector:
    env: prod
    disk: ssd
```

### Meaning

```text
env=prod AND disk=ssd
```

All labels must match.

### Result

```text
✅ Node1
❌ Node2
❌ Node3
```

---

## How Node Selector Works

Node Selector performs an exact label match:

```text
key = value
```

Example:

```yaml
nodeSelector:
  env: prod
```

Meaning:

```text
env = prod
```

---

## Limitation of Node Selector

Node Selector only supports:

```text
key = value
```

It cannot express:

```text
env IN (prod, qa)
env NOT IN (dev)
region EXISTS
cpu > 4
```

For these advanced requirements, use Node Affinity.

---

# Node Selector vs Node Affinity

## Same Requirement Using Node Selector

```yaml
nodeSelector:
  zone: az1
```

Meaning:

```text
zone = az1
```

---

## Same Requirement Using Node Affinity

```yaml
requiredDuringSchedulingIgnoredDuringExecution:
  nodeSelectorTerms:
  - matchExpressions:
    - key: zone
      operator: In
      values:
      - az1
      - az2
```

Meaning:

```text
zone = az1 OR az2
```

---

## Comparison

| Node Selector | Node Affinity |
|---------------|---------------|
| Simple | Advanced |
| Exact label matching | Supports complex rules |
| Hard requirement only | Hard and soft requirements |
| Easy to configure | More flexible |
| key=value | In, NotIn, Exists, DoesNotExist, Gt, Lt |

---

## When to Use

### Use Node Selector When

- Exact node label matching is sufficient.
- Scheduling requirements are simple.
- No preference rules are needed.

### Use Node Affinity When

- Multiple acceptable values are needed.
- Preferred nodes should be prioritized.
- Advanced scheduling logic is required.

---

## Memory Trick

```text
Node Selector = Run only on this node label

Node Affinity = Run on these node labels and optionally prefer some nodes over others
```

---

## Key Takeaway

- Node Selector is the simplest scheduling constraint.
- It uses exact label matching (`key=value`).
- If no matching node exists, the Pod remains Pending.
- Node Affinity is a more powerful and flexible alternative.