# Kubernetes Volumes

## Why Volumes Exist

Containers have an ephemeral filesystem.

Example:

```text
Container Starts
      ↓
Creates File
      ↓
Container Restarts
      ↓
File Lost
```

A Volume provides storage that is independent of a container's lifecycle.

```text
Container
     ↓
Writes Data
     ↓
Volume
     ↓
Data Persists
```

---

# Core Concepts

## Volume

A Volume is a storage resource attached to a Pod.

```yaml
volumes:
- name: app-data
```

Think:

```text
Volume = Storage
```

---

## VolumeMount

A VolumeMount tells a container where to access that storage.

```yaml
volumeMounts:
- name: app-data
  mountPath: /data
```

Think:

```text
VolumeMount = Access Path
```

---

## Relationship

```text
Volume
   ↓
Storage Resource

VolumeMount
   ↓
Path Inside Container
```

---

# Pod, Volume and Container Relationship

```text
Pod
│
├── Volume
│   └── app-data
│
├── Container A
│   └── /data
│
└── Container B
    └── /shared
```

Both containers access the same storage.

---

# Example: Shared Volume

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: shared-volume

spec:

  volumes:
  - name: shared-storage
    emptyDir: {}

  containers:

  - name: writer
    image: busybox

    volumeMounts:
    - name: shared-storage
      mountPath: /logs

  - name: reader
    image: busybox

    volumeMounts:
    - name: shared-storage
      mountPath: /shared
```

---

## Result

Writer:

```text
/logs/app.log
```

Reader:

```text
/shared/app.log
```

Same file.

Same volume.

Different mount paths.

---

# Types of Volumes

Instead of memorizing types, classify them by where the data lives.

---

# 1. emptyDir

## Where is data stored?

```text
Node
 └── Temporary Directory
```

created automatically by Kubernetes.

---

## Lifecycle

```text
Pod Starts
      ↓
emptyDir Created
      ↓
Pod Deleted
      ↓
emptyDir Deleted
```

---

## Example

```yaml
volumes:
- name: cache
  emptyDir: {}
```

---

## Use Cases

### Sidecar Logging

```text
App Container
     ↓ writes logs

emptyDir

Log Collector Sidecar
     ↓ reads logs
```

---

### Temporary Cache

```text
Download Files
      ↓
Process Files
      ↓
Delete Pod
```

No persistence needed.

---

## Important

Container restart:

```text
Container Restart
       ↓
Data Survives
```

Pod deletion:

```text
Pod Deleted
      ↓
Data Lost
```

---

# 2. hostPath

## Where is data stored?

```text
Node Filesystem
```

Example:

```text
Node1
└── /data
```

---

## Example

```yaml
volumes:
- name: host-storage
  hostPath:
    path: /data
```

---

## Result

```text
Node1:/data
        ↓
Container:/app-data
```

---

## Use Cases

### Access Node Logs

```text
Node
└── /var/log

Pod
└── /logs
```

---

### Monitoring Agents

Prometheus Node Exporter

Fluent Bit

Filebeat

Often need access to:

```text
/var/log
/proc
/sys
```

on the host.

---

## Problem

Suppose:

```text
Pod Running On Node1
```

Data:

```text
Node1:/data/app.db
```

Node1 crashes:

```text
Scheduler
     ↓
Moves Pod To Node2
```

Node2:

```text
/data/app.db
```

does not exist.

Application breaks.

---

## Key Point

```text
hostPath = Node Dependent
```

---

# 3. Persistent Volume (PV)

A Persistent Volume represents actual storage available in the cluster.

Examples:

```text
AWS EBS
Azure Disk
Google Persistent Disk
NFS
Ceph
Local Storage
```

Think:

```text
PV = Actual Storage Resource
```

---

# 4. Persistent Volume Claim (PVC)

A PVC is a request for storage.

Example:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim

spec:
  accessModes:
  - ReadWriteOnce

  resources:
    requests:
      storage: 10Gi
```

---

## Think Of It Like

```text
PVC = Storage Request

PV = Storage Resource
```

---

## Flow

```text
Pod
 ↓
PVC
 ↓
PV
 ↓
Disk
```

---

# Example

## PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim

metadata:
  name: mysql-pvc

spec:
  accessModes:
  - ReadWriteOnce

  resources:
    requests:
      storage: 10Gi
```

---

## Pod

```yaml
volumes:
- name: mysql-data

  persistentVolumeClaim:
    claimName: mysql-pvc
```

---

## Result

```text
MySQL Pod
      ↓
PVC
      ↓
Persistent Disk
```

---

# Why PVC Exists

Without PVC:

```text
Application
      ↓
Needs To Know
AWS EBS
NFS
Ceph
```

With PVC:

```text
Application
      ↓
Requests Storage
      ↓
Kubernetes Provides It
```

Storage becomes portable.

---

# hostPath vs PVC

## hostPath

```text
Pod
 ↓
Node1:/data
```

Storage tied to node.

---

## PVC

```text
Pod
 ↓
PVC
 ↓
Persistent Storage
```

Storage independent of pod.

---

## Real World Analogy

### hostPath

```text
Save Files On Laptop
```

Laptop dies:

```text
Files Gone
```

---

### PVC

```text
Save Files On Google Drive
```

Laptop dies:

```text
Login From New Laptop
      ↓
Files Still Available
```

---

# ConfigMap Volume

ConfigMap can be mounted as files.

ConfigMap:

```yaml
data:
  app.properties: |
    env=dev
```

Mount:

```yaml
volumes:
- name: config-volume

  configMap:
    name: app-config
```

---

## Result

```text
/etc/config/app.properties
```

inside container.

---

# Secret Volume

Secret:

```yaml
kind: Secret
```

Mount:

```yaml
volumes:
- name: secret-volume

  secret:
    secretName: db-secret
```

---

## Result

```text
/etc/secrets/password
```

inside container.

---

# Storage Hierarchy

```text
Container
    ↓
VolumeMount
    ↓
Volume
    ↓
Storage Backend

Examples:

emptyDir
hostPath
ConfigMap
Secret
PVC
```

---

# Interview Questions

## Difference Between Volume And VolumeMount

```text
Volume
    = Storage Resource

VolumeMount
    = Access Path Inside Container
```

---

## Difference Between hostPath And PVC

hostPath:

```text
Uses Specific Node Storage
```

PVC:

```text
Uses Kubernetes Persistent Storage
```

---

## Does Volume Survive Container Restart?

```text
Yes
```

Volume belongs to Pod.

---

## Does emptyDir Survive Pod Deletion?

```text
No
```

---

## Does PVC Survive Pod Deletion?

```text
Yes
```

Storage remains.

---

# Memory Tricks

Volume

```text
Storage
```

VolumeMount

```text
Path To Storage
```

---

hostPath

```text
Node Storage
```

PVC

```text
Cluster Storage
```

---

emptyDir

```text
Temporary Storage
```

---

ConfigMap Volume

```text
Configuration Files
```

---

Secret Volume

```text
Sensitive Files
```