# **Kubernetes on Minikube**

## **Architecture Overview**

```text
Your Mac
│
└── Minikube
    │
    └── Kubernetes Cluster
        ├── API Server
        ├── Scheduler
        ├── Controller Manager
        ├── etcd
        └── Worker Nodes
```

When using the Docker driver, Minikube runs Kubernetes nodes as Docker containers on your machine.

---

## **Why Minikube?**

Instead of provisioning cloud infrastructure on:

- Amazon Web Services (AWS)
- Google Cloud Platform (GCP)
- Microsoft Azure

you can run a complete Kubernetes cluster locally on your laptop for learning, testing, and development.

Benefits:

- No cloud cost
- Fast setup
- Easy experimentation
- Ideal for learning Kubernetes concepts

---

# **Installing Kubernetes Tools**

## **Install Minikube**

```bash
brew install minikube
```

Minikube creates and manages a local Kubernetes cluster.

---

## **Install kubectl**

```bash
brew install kubernetes-cli
```

This installs:

```bash
kubectl
```

`kubectl` is the Kubernetes command-line tool used to interact with the cluster.

Example:

```bash
kubectl get nodes
kubectl get pods -A
kubectl describe node minikube
```

---

## **Minikube vs kubectl**

Think of them as separate responsibilities:

```text
Minikube
└── Creates and manages the cluster

kubectl
└── Communicates with the cluster
```

Flow:

```text
You
 │
 ▼
kubectl
 │
 ▼
Kubernetes API Server
 │
 ▼
Cluster Resources
```

---

# **Minikube Drivers**

A driver is the technology Minikube uses to create Kubernetes nodes.

Common drivers:

|**Driver**|**Creates**|
|---|---|
|Docker|Containers|
|HyperKit|Virtual Machines|
|VirtualBox|Virtual Machines|
|QEMU|Virtual Machines|

Since Docker Desktop is installed:

```text
Minikube
└── Docker Driver
```

is selected automatically.

---

# **Container Runtime**

Minikube currently supports multiple container runtimes.

Examples:

- Docker
- containerd

Recent Minikube versions indicate:

```text
Minikube will default to containerd in future releases.
```

Why?

```text
Docker
└── containerd
```

Docker itself uses containerd internally.

Modern Kubernetes interacts directly with containerd, making the architecture simpler.

---

# **Minikube, KICBase & Debian**

## **The Common Confusion**

Docker shows:

```bash
docker ps
```

Output:

```text
IMAGE
gcr.io/k8s-minikube/kicbase:v0.0.50
```

Kubernetes shows:

```bash
kubectl get nodes -o wide
```

Output:

```text
OS-IMAGE
Debian GNU/Linux 12 (bookworm)
```

At first glance it looks like Minikube is using two different images.

It is not.

---

## **Relationship Between KICBase and Debian**

```text
KICBase Image
│
└── Debian 12
    ├── Networking Tools
    ├── Container Runtime Components
    ├── Kubernetes Dependencies
    └── Node Bootstrap Tooling
```

KICBase is a Docker image created by the Minikube team.

Debian is the operating system installed inside that image.

---

## **Different Perspectives**

### **Docker Perspective**

Docker reports:

```text
kicbase:v0.0.50
```

because it shows the image used to create the container.

---

### **Kubernetes Perspective**

Kubernetes reports:

```text
Debian GNU/Linux 12
```

because it shows the operating system running inside the node.

---

## **Analogy**

```text
MacBook Air
└── macOS
```

Similarly:

```text
KICBase
└── Debian 12
```

- MacBook Air = Hardware/Product
- macOS = Operating System

Likewise:

- KICBase = Container Image
- Debian = Operating System inside the image

---

## **Key Takeaway**

Docker and Kubernetes are looking at the same node from different perspectives:

```text
Docker:
"What image created this container?"
→ KICBase

Kubernetes:
"What operating system is running?"
→ Debian 12
```

Therefore:

```text
KICBase Image
└── Debian 12 OS
```

Both refer to the same Minikube node.

---


# **Minikube Networking & Architecture Notes**

## **SSH Into Minikube Nodes**

Minikube provides direct SSH access to the nodes.

```bash
# Control Plane
minikube ssh

# Specific Worker Node
minikube ssh -n minikube-m02
```

---

## **Exposed Ports on a Minikube Node**

Example from Docker:

```text
127.0.0.1:57101 -> 22/tcp
127.0.0.1:57102 -> 2376/tcp
127.0.0.1:57100 -> 5000/tcp
127.0.0.1:57104 -> 8443/tcp
127.0.0.1:57103 -> 32443/tcp
```

Common ports:

|**Port**|**Purpose**|
|---|---|
|22|SSH Access|
|2376|Docker Daemon|
|5000|Internal Registry|
|8443|Kubernetes API Server|
|32443|Kubernetes Service / NodePort|

---

## **Handy Commands**

```bash
kubectl describe node minikube

docker network inspect minikube

kubectl get pods -A -o wide
```

---

# **Minikube Network Overview**

```bash
admin@MacBook-Air ~/D/SRE-Bootcamp (ci/cd)> docker network inspect minikube
[
    {
        "Name": "minikube",
        "Id": "ae580e5966afd67eacaf4ac372b6454f1afd42ce54d6e842e92d3f3407e4618c",
        "Created": "2026-07-09T11:17:53.571739542Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv4": true,
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.49.0/24",
                    "Gateway": "192.168.49.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Options": {
            "--icc": "",
            "--ip-masq": "",
            "com.docker.network.driver.mtu": "65535",
            "com.docker.network.enable_ipv4": "true",
            "com.docker.network.enable_ipv6": "false"
        },
        "Labels": {
            "created_by.minikube.sigs.k8s.io": "true",
            "name.minikube.sigs.k8s.io": "minikube"
        },
        "Containers": {
            "373fea076d84e7f4fad7863f6e6f31a63b052208ce0f9970e0bf2d7a4b2a9c7a": {
                "Name": "minikube",
                "EndpointID": "f19e3ea6c0c4fc1866d1a1a4c0e08cab277fd95d0ab8e70e953ad85abc72f786",
                "MacAddress": "ba:e6:93:2c:e0:b2",
                "IPv4Address": "192.168.49.2/24",
                "IPv6Address": ""
            },
            "7a0d07b5c66ade63398cceba606615968be0bae55e89749f82b756e83b3b9d9e": {
                "Name": "minikube-m03",
                "EndpointID": "b65bfa7ed8d5236ae5d58b9fff87ec561fb94e8be32beb7fb8381667d37e4f7a",
                "MacAddress": "fa:46:88:4f:b8:e9",
                "IPv4Address": "192.168.49.4/24",
                "IPv6Address": ""
            },
            "d0b8ca31bb48c19c6874d8e00636278fc32131f277b0e64a6e3223b113578a32": {
                "Name": "minikube-m04",
                "EndpointID": "861f6efee4302f4b1f1b8dec73e0ae23094e76e858c43a80626f739561bed6f0",
                "MacAddress": "0e:15:1a:bd:8a:43",
                "IPv4Address": "192.168.49.5/24",
                "IPv6Address": ""
            },
            "eec09ef713395c0665fe19d02964899b3714a3c54ac720d6fbe36b0471e1dc5e": {
                "Name": "minikube-m02",
                "EndpointID": "abfb1948ec0cd5db54f7dc58ef7a67109d33d52fab931936420ece9211015912",
                "MacAddress": "c2:bb:73:4e:d9:8b",
                "IPv4Address": "192.168.49.3/24",
                "IPv6Address": ""
            }
        },
        "Status": {
            "IPAM": {
                "Subnets": {
                    "192.168.49.0/24": {
                        "IPsInUse": 7,
                        "DynamicIPsAvailable": 249
                    }
                }
            }
        }
    }
]
```

Cluster:

```text
1 Control Plane
3 Worker Nodes
```

Docker Network:

```text
Network Name : minikube
Driver       : bridge
Subnet       : 192.168.49.0/24
Gateway      : 192.168.49.1
```

### **Node IP Allocation**

```text
192.168.49.1  → Docker Bridge Gateway

192.168.49.2  → minikube (Control Plane)
192.168.49.3  → minikube-m02
192.168.49.4  → minikube-m03
192.168.49.5  → minikube-m04
```

Visualization:

```text
Docker Bridge Network
192.168.49.0/24

                 ┌─────────────────┐
                 │ Gateway (.1)    │
                 └────────┬────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
        ▼                 ▼                 ▼

   minikube         minikube-m02      minikube-m03
192.168.49.2      192.168.49.3      192.168.49.4

                          │
                          ▼

                    minikube-m04
                   192.168.49.5
```

---

## **Understanding the Subnet**

```text
192.168.49.0/24
```

A `/24` network means:

```text
Network Address : 192.168.49.0
Gateway         : 192.168.49.1
Usable Range    : 192.168.49.1 - 192.168.49.254
Broadcast       : 192.168.49.255
```

Approximately **254 usable IP addresses**.

---

## **Understanding the Gateway**

Docker reserves:

```text
192.168.49.1
```

for the bridge gateway.

The gateway acts as the router for the Docker network.

Responsibilities:

- Connect containers to each other
- Connect containers to the Docker host
- Connect containers to the Internet

Think of it as:

```text
Home Network
├── Laptop
├── Mobile
├── TV
└── Router (Gateway)
```

Similarly:

```text
Minikube Network
├── minikube
├── minikube-m02
├── minikube-m03
├── minikube-m04
└── Gateway (192.168.49.1)
```

---

## **IP Address vs MAC Address**

Every network interface receives:

### **IP Address (Layer 3)**

Example:

```text
192.168.49.2
```

Used for routing packets across networks.

Analogy:

```text
IP Address = House Address
```

---

### **MAC Address (Layer 2)**

Example:

```text
ba:e6:93:2c:e0:b2
```

Used for communication within the same network.

Analogy:

```text
MAC Address = Person's Identity Card
```

Docker uses MAC addresses internally to deliver frames between containers connected to the bridge network.

---

# **How kubectl Communicates**

## **Does kubectl Talk to the Gateway?**

No.

Normally, kubectl communicates directly with the Kubernetes API Server.

Flow:

```text
You
 │
 ▼
kubectl
 │
 ▼
~/.kube/config
 │
 ▼
Kubernetes API Server
 │
 ▼
etcd / Scheduler / Controllers
```

The gateway is not involved in this communication path.

---

## **Actual Request Flow**

Example:

```bash
kubectl get nodes
```

Flow:

```text
You
 │
 ▼
kubectl
 │
 ▼
~/.kube/config
 │
 ▼
API Server Endpoint
 │
 ▼
Control Plane Node
(192.168.49.2)
 │
 ▼
Node Information Returned
 │
 ▼
Terminal Output
```

---

# **When Does the Gateway Get Used?**

The gateway becomes involved when traffic leaves the Docker network.

Examples:

### **Pulling Container Images**

```text
Worker Node
192.168.49.3
      │
      ▼
Gateway
192.168.49.1
      │
      ▼
Internet
      │
      ▼
Docker Hub
```

Example:

```bash
kubectl apply -f nginx.yaml
```

The node may need to pull:

```text
nginx:latest
```

from Docker Hub.

The traffic exits through the gateway.

---

### **Accessing External Services**

Examples:

- Docker Hub
- GitHub
- AWS APIs
- External Databases
- Public APIs

All of these use the gateway.

---

# **Simple Analogy**

```text
Mac                = Apartment Building
Docker Network     = Floor
Gateway            = Security Desk
Minikube Nodes     = Apartments
kubectl            = You calling an apartment
```

### **Normal kubectl Request**

```text
You
 │
 ▼
Apartment 202
```

No security desk involved.

---

### **Accessing the Internet**

```text
Apartment 202
 │
 ▼
Security Desk
 │
 ▼
Outside World
```

The gateway is now involved.

---

# **Key Takeaways**

- Each Minikube node is a Docker container.
- All nodes are attached to the same Docker bridge network.
- Docker created the subnet `192.168.49.0/24`.
- Docker reserved `192.168.49.1` as the gateway.
- Nodes communicate using their assigned IP addresses.
- `kubectl` talks directly to the Kubernetes API Server.
- The gateway is mainly used when nodes need to communicate outside the Docker network.
- Common gateway use cases:
    - Pulling container images
    - Accessing the Internet
    - Reaching external services
    - Communicating across Docker networks
