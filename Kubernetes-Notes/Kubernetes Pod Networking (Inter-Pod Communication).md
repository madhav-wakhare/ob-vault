# Kubernetes Pod Networking (Inter-Pod Communication)

## Why Learn This?

Many beginners think:

```text
Pod A
   ↓
Directly talks to
   ↓
Pod B
```

But internally there are several networking components involved.

Understanding this helps with:

- Kubernetes Networking
- CNI (Calico, Flannel, Cilium)
- Services
- Network Policies
- Troubleshooting Pod Connectivity

---

# The Biggest Misconception

❌ Wrong

```text
Pods communicate using root filesystem (rootfs)
```

Root filesystem has nothing to do with networking.

---

## What is RootFS?

RootFS is simply the container's filesystem.

Example:

```text
/
├── bin
├── etc
├── home
├── usr
├── var
└── tmp
```

Think:

```text
RootFS = Hard Disk
```

Purpose:

```text
Store Files
Store Configurations
Store Logs
Store Application Code
```

Example:

```text
/etc/nginx/nginx.conf
```

lives in RootFS.

---

## What is NOT in RootFS?

Networking.

RootFS does NOT provide:

```text
IP Address
Network Interface
Routing Table
Network Connectivity
```

---

# What Actually Enables Networking?

## Network Namespace

Every Pod gets its own Network Namespace.

Think:

```text
Network Namespace = Private Network Room
```

Inside that room, the Pod has:

```text
eth0
lo
Routing Table
ARP Table
```

Just like a small Linux machine.

---

# House Analogy

Imagine an apartment building.

## RootFS

```text
Apartment Interior
```

Contains:

```text
Furniture
TV
Kitchen
Bed
```

Equivalent to:

```text
Files
Directories
Application Code
```

---

## Network Namespace

```text
Apartment Door
Address
Letterbox
```

Equivalent to:

```text
IP Address
eth0
Routing
```

This allows communication.

---

# What Happens When Kubernetes Creates a Pod?

Suppose:

```text
kubectl apply -f pod.yaml
```

---

## Step 1

Kubernetes creates:

```text
Pod Network Namespace
```

Think:

```text
Private Network Room
```

---

## Step 2

CNI creates a veth pair.

---

# What is a veth Pair?

Think of a veth pair as:

```text
A Network Cable
```

with two ends.

```text
End A <-------> End B
```

Anything entering one side exits the other.

---

## Example

```text
Pod Side            Host Side

eth0 <---------> veth1234
```

---

One end:

```text
eth0
```

stays inside Pod.

---

Other end:

```text
veth1234
```

stays on Node.

---

# Visual Representation

```text
Pod Network Namespace

eth0
  │
  │
  ▼

veth Pair

  ▲
  │
  │

veth1234

Host Network Namespace
```

---

# How Two Pods Communicate On Same Node

Suppose:

```text
Node1
```

contains:

```text
Pod A
Pod B
```

---

## Network Layout

```text
Pod A
  eth0
    │
    ▼
  vethA
    │
    ▼

      cni0 Bridge

    ▲
    │
  vethB
    ▲
    │

Pod B
  eth0
```

---

# What is cni0?

Think:

```text
cni0 = Network Switch
```

inside the Node.

---

Real World Analogy

```text
Laptop A
Laptop B
Laptop C

      │
      │
      ▼

Network Switch
```

The switch forwards traffic.

---

Kubernetes does the same.

```text
Pod A
Pod B
Pod C

      │
      ▼

cni0 Bridge
```

---

# Traffic Flow

Suppose:

```text
Pod A
IP: 10.244.0.2
```

wants to talk to:

```text
Pod B
IP: 10.244.0.3
```

---

Packet flow:

```text
Pod A eth0
      ↓
vethA
      ↓
cni0 Bridge
      ↓
vethB
      ↓
Pod B eth0
```

---

# Different Node Communication

Suppose:

```text
Node1
```

contains:

```text
Pod A
```

and

```text
Node2
```

contains:

```text
Pod B
```

---

## Layout

```text
Node1                     Node2

Pod A                     Pod B

eth0                      eth0
 │                          ▲
 ▼                          │

vethA                    vethB

 │                          ▲
 ▼                          │

CNI Network Routing
```

---

# How Does Traffic Reach Another Node?

This depends on CNI.

Examples:

- Calico
- Flannel
- Cilium

---

The CNI provides:

```text
Routing
Overlay Networking
Encapsulation
Packet Forwarding
```

---

# Example With Calico

Suppose:

```text
Pod A
10.244.0.2
```

wants to reach:

```text
Pod B
10.244.1.2
```

Calico knows:

```text
10.244.1.0/24
      ↓
Node2
```

and routes traffic there.

---

# Kubernetes Networking Rule

One of the most important interview concepts:

Every Pod can communicate with every other Pod using its IP.

Without:

```text
NAT
Port Mapping
Manual Routes
```

---

This is called:

```text
Flat Pod Network
```

---

# Complete Packet Journey

Same Node:

```text
Pod A
  ↓
eth0
  ↓
vethA
  ↓
cni0
  ↓
vethB
  ↓
eth0
  ↓
Pod B
```

---

Different Nodes:

```text
Pod A
  ↓
eth0
  ↓
vethA
  ↓
Node1
  ↓
CNI Network
  ↓
Node2
  ↓
vethB
  ↓
eth0
  ↓
Pod B
```

---

# Components Summary

## RootFS

Purpose:

```text
File Storage
```

Contains:

```text
/bin
/etc
/var
/usr
```

Analogy:

```text
Apartment Interior
```

---

## Network Namespace

Purpose:

```text
Network Isolation
```

Contains:

```text
eth0
lo
Routing Table
```

Analogy:

```text
Apartment Address
```

---

## veth Pair

Purpose:

```text
Connect Pod To Node
```

Analogy:

```text
Network Cable
```

---

## cni0 Bridge

Purpose:

```text
Connect Pods On Same Node
```

Analogy:

```text
Network Switch
```

---

## CNI

Purpose:

```text
Connect Pods Across Cluster
```

Examples:

```text
Calico
Flannel
Cilium
Weave
```

Analogy:

```text
Road Network Between Cities
```

---

# Interview Questions

## Does rootfs enable Pod networking?

No.

RootFS only provides filesystem storage.

Networking is provided through:

```text
Network Namespace
eth0
veth Pair
CNI
```

---

## What is a veth Pair?

A virtual ethernet cable with two connected ends.

One end stays inside the Pod.

One end stays on the Node.

---

## What is cni0?

A Linux bridge created by many CNIs that acts like a network switch for Pods on the same Node.

---

## How do Pods communicate on the same Node?

```text
Pod
 ↓
eth0
 ↓
veth
 ↓
cni0
 ↓
veth
 ↓
eth0
 ↓
Other Pod
```

---

## How do Pods communicate across Nodes?

```text
Pod
 ↓
veth
 ↓
Node
 ↓
CNI Routing
 ↓
Other Node
 ↓
veth
 ↓
Other Pod
```

---

# Memory Tricks

```text
RootFS = Hard Disk
```

```text
Network Namespace = Private Network Room
```

```text
veth Pair = Network Cable
```

```text
cni0 = Network Switch
```

```text
CNI = Road Network
```

```text
Pod IP = House Address
```

---

# Final Mental Model

```text
Pod A

Files
  ↓
RootFS

Networking
  ↓
eth0
  ↓
veth Pair
  ↓
cni0 / CNI Network
  ↓
veth Pair
  ↓
eth0

Pod B
```

**RootFS stores files.  
Network Namespace + veth + CNI enable communication.**