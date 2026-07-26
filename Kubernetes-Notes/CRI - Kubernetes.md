# **CRI (Container Runtime Interface)**

## **What is CRI?**

CRI (**Container Runtime Interface**) is a **standard/API contract**, not a software.

It defines how Kubernetes communicates with a container runtime.

Think of CRI as a common language between **Kubelet** and container runtimes.

---

## **Why CRI Exists?**

Without CRI, Kubernetes would need separate code for every runtime.

```text
Kubelet
 ├── Docker API
 ├── containerd API
 ├── CRI-O API
 └── Runtime-specific logic
```

This would be difficult to maintain.

CRI provides a common interface:

```text
Kubelet
    ↓
CRI
    ↓
containerd / CRI-O
```

Now Kubernetes can work with multiple runtimes without knowing their internal implementation.

---

# **CRI = Standard, Not Software**

A very common confusion.

### **CRI**

```text
Standard
Contract
Interface
Specification
```

### **containerd / CRI-O**

```text
Software
Implementations
Runtime Engines
```

### **Analogy**

```text
SQL
 ↓
Standard

MySQL
PostgreSQL
MariaDB
 ↓
Implementations
```

Similarly:

```text
CRI
 ↓
Standard

containerd
CRI-O
 ↓
Implementations
```

---

# **Important Clarification About Docker**

Many people assume Docker implements CRI.

This is incorrect.

### **containerd**

```text
Native CRI Support
```

### **CRI-O**

```text
Native CRI Support
```

### **Docker Engine**

```text
No Native CRI Support
```

Docker uses its own API.

To use Docker with Kubernetes:

```text
Kubelet
   ↓
CRI
   ↓
cri-dockerd
   ↓
Docker Engine
```

`cri-dockerd` acts as a translator.

---

# **Responsibilities of CRI**

CRI defines operations such as:

```text
PullImage()
CreateContainer()
StartContainer()
StopContainer()
RemoveContainer()
GetContainerStatus()
```

Important:

CRI does **not** create containers.

It only defines how Kubernetes requests those operations.

The runtime performs the actual work.

---

# **Pod Creation Flow**

When a Pod is deployed:

```text
kubectl apply
      ↓
API Server
      ↓
Scheduler
      ↓
Selected Node
      ↓
Kubelet
```

Kubelet is responsible for ensuring the Pod runs on the node.

---

# **Container Creation Flow**

Kubelet does not create containers directly.

It uses CRI to communicate with the runtime.

```text
Kubelet
    ↓
CRI
    ↓
containerd
    ↓
containerd-shim
    ↓
runc
    ↓
Container
```

---

# **Purpose of Each Component**

## **Kubelet**

Node agent running on every Kubernetes node.

Responsibilities:

- Watches assigned Pods
- Talks to container runtime via CRI
- Talks to CNI for networking
- Reports node health to API Server

---

## **CRI**

Container Runtime Interface.

Responsibilities:

- Define runtime communication standard
- Abstract runtime implementation details
- Allow Kubernetes to support multiple runtimes

CRI itself performs no container operations.

---

## **containerd**

Container runtime.

Responsibilities:

- Pull images
- Manage containers
- Manage container lifecycle

Examples:

```text
Pull Image
Create Container
Start Container
Stop Container
Delete Container
```

---

## **containerd-shim**

Intermediate process between containerd and containers.

Responsibilities:

- Keeps containers running if containerd restarts
- Handles container stdin/stdout
- Monitors container process

Without shim:

```text
containerd crash
      ↓
containers stop
```

---

## **runc**

OCI-compliant low-level runtime.

Responsibilities:

- Create Linux namespaces
- Create cgroups
- Launch container process

Think of runc as:

```text
The component that actually starts the container.
```

---

# **CRI vs CNI**

A common interview question.

|**Component**|**Purpose**|
|---|---|
|CRI|Container Management|
|CNI|Networking|

### **CRI Answers**

```text
How do I create and run a container?
```

Examples:

- containerd
- CRI-O

---

### **CNI Answers**

```text
How does the Pod communicate?
```

Examples:

- Flannel
- Calico
- Cilium

---

# **CRI vs CNI Analogy**

```text
CRI
 ↓
Runtime Standard

containerd
CRI-O
```

```text
CNI
 ↓
Networking Standard

Flannel
Calico
Cilium
```

Both are standards.

The actual work is performed by their implementations.

---

# **CRI, CNI & OCI Relationship**

Kubernetes relies on multiple standards.

```text
Kubernetes
     │
     ├── CRI
     │      ↓
     │  containerd / CRI-O
     │
     ├── CNI
     │      ↓
     │  Flannel / Calico / Cilium
     │
     └── OCI
            ↓
          runc
```

---

## **Purpose of Each Standard**

|**Standard**|**Purpose**|
|---|---|
|CRI|Runtime Communication|
|CNI|Networking|
|OCI|Container Execution & Image Format|

---

# **Complete Kubernetes Node Flow**

```text
Pod Scheduled
      ↓
Kubelet
      │
      ├── CRI
      │      ↓
      │  containerd
      │      ↓
      │  containerd-shim
      │      ↓
      │     runc
      │      ↓
      │  Container Started
      │
      └── CNI
              ↓
      Create Interface
              ↓
         Assign IP
              ↓
      Configure Routes
              ↓
       Networking Ready
```

---

# **Memory Tricks**

### **CRI**

```text
CRI = Runtime Communication
```

### **CNI**

```text
CNI = Networking
```

### **OCI**

```text
OCI = Container Execution Standard
```

---

# **Summary (Understand Everything in One Go)**

## **High-Level View**

```text
Kubernetes
     │
     ▼
Kubelet
     │
     ├── CRI
     │      ↓
     │  containerd
     │      ↓
     │  containerd-shim
     │      ↓
     │     runc
     │      ↓
     │  Container Running
     │
     └── CNI
             ↓
      Network Interface
             ↓
         Pod IP
             ↓
      Pod Connectivity
```

### **What Each Component Does**

```text
Kubelet
→ Node agent
→ Orchestrates Pod lifecycle

CRI
→ Standard for runtime communication

containerd
→ Container runtime
→ Manages images and containers

containerd-shim
→ Keeps containers alive
→ Bridges containerd and container

runc
→ Actually launches container process

CNI
→ Networking standard

Flannel / Calico / Cilium
→ Implement CNI
→ Assign IPs and routes

OCI
→ Standard for container execution
```

### **Final Mental Model**

```text
CRI
→ "Run the container"

containerd
→ "Manage the container"

runc
→ "Start the container"

CNI
→ "Connect the container"

OCI
→ "Define how containers should run"
```

### **Interview One-Liner**

```text
CRI is a standard that allows Kubelet to communicate with container runtimes such as containerd and CRI-O.

CNI is a standard that allows Kubernetes to configure networking using plugins such as Flannel, Calico, and Cilium.

OCI is a standard that defines how containers are packaged and executed, with runtimes like runc implementing it.
```