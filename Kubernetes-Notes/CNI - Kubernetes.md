# **CNI (Container Network Interface)**

## **What is CNI?**

CNI (**Container Network Interface**) is a **specification/standard**, not a networking software.

It defines how container runtimes and orchestrators (like Kubernetes) should configure networking for containers and Pods.

### **Analogy**

```text
SQL Standard
      ↓
MySQL
PostgreSQL
MariaDB

CNI Standard
      ↓
Flannel
Calico
Cilium
```

The standard defines the rules.

Implementations provide the actual functionality.

---

# **Why Do We Need CNI?**

When a Pod is created, Kubernetes must:

- Create a network interface
- Assign an IP address
- Configure routes
- Enable Pod-to-Pod communication

Kubernetes itself does not perform these tasks.

It delegates them to a CNI plugin.

---

# **Popular CNI Implementations**

|**CNI**|**Purpose**|
|---|---|
|Flannel|Simple Pod networking|
|Calico|Networking + Network Policies|
|Cilium|eBPF-based networking, security & observability|
|Weave|Overlay networking|
|Antrea|Open vSwitch based networking|

---

# **Responsibilities of a CNI Plugin**

When a Pod is created:

```text
Create Network Interface
        ↓
Assign Pod IP
        ↓
Configure Routes
        ↓
Enable Connectivity
```

Without CNI:

```text
Pod Exists
      ↓
No IP Address
      ↓
No Networking
```

---

# **Container Runtime vs CNI**

|**Component**|**Responsibility**|
|---|---|
|Container Runtime|Create and run containers|
|CNI|Configure networking for containers|

### **Container Runtime**

Answers:

```text
How do I run a container?
```

Examples:

- containerd
- CRI-O
- Docker (via cri-dockerd)

---

### **CNI**

Answers:

```text
How does the Pod communicate?
```

Examples:

- Flannel
- Calico
- Cilium

---

# **Kubernetes Pod Creation Flow**

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

Kubelet is the primary node agent responsible for making sure the Pod runs.

---

# **Container Creation Flow**

Kubelet does not create containers directly.

It talks to the Container Runtime using the CRI (Container Runtime Interface).

```text
Kubelet
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

## **Purpose of Each Component**

### **Kubelet**

Node agent running on every Kubernetes node.

Responsibilities:

- Watches for Pods assigned to the node
- Talks to container runtime
- Talks to CNI plugin
- Reports node status

---

### **containerd**

Container runtime.

Responsibilities:

- Pull images
- Manage containers
- Manage container lifecycle

Example:

```text
Pull nginx image
Create container
Start container
```

---

### **containerd-shim**

Intermediate process between containerd and containers.

Responsibilities:

- Keeps containers running even if containerd restarts
- Handles container stdin/stdout
- Manages container lifecycle

Without shim:

```text
containerd crash
      ↓
containers stop
```

---

### **runc**

Low-level OCI runtime.

Responsibilities:

- Creates Linux namespaces
- Creates cgroups
- Starts container process

Think of runc as:

```text
The component that actually launches the container.
```

---

# **Complete Pod Startup Flow**

```text
Pod Scheduled
      ↓
Kubelet
      ↓
containerd
      ↓
containerd-shim
      ↓
runc
      ↓
Container Started
```

---

# **Networking Flow (CNI)**

After the container starts:

```text
Kubelet
      ↓
CNI Plugin
      ↓
Create Interface
      ↓
Assign IP
      ↓
Configure Routes
      ↓
Pod Networking Ready
```

Example:

```text
Kubelet
      ↓
Flannel
      ↓
eth0 created
      ↓
10.244.0.5 assigned
      ↓
Pod can communicate
```

---

# **Complete Kubernetes Flow**

```text
kubectl apply
      ↓
API Server
      ↓
Scheduler
      ↓
Node Selected
      ↓
Kubelet
      ├── containerd
      │      ↓
      │  containerd-shim
      │      ↓
      │     runc
      │      ↓
      │  Container Started
      │
      └── CNI Plugin
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

### **Container Runtime**

```text
Runtime gives the Pod life.
```

### **CNI**

```text
CNI gives the Pod connectivity.
```

### **Easy Interview Answer**

```text
Container Runtime:
Creates and runs containers.

CNI:
Provides networking to Pods.

CNI is a standard.

Flannel, Calico, and Cilium are implementations of that standard.
```