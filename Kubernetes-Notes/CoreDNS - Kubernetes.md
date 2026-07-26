# **Kubernetes Service Discovery: Service & CoreDNS**

Pods in Kubernetes are ephemeral and their IP addresses can change whenever they are recreated.

To ensure reliable communication between applications, Kubernetes provides **Service Discovery** using two components:

- **Service** → Provides a stable virtual IP and routes traffic to the correct Pods.
- **CoreDNS** → Resolves Service names to their corresponding Service IPs.

Together they allow applications to communicate using names instead of changing Pod IPs.

```text
Application Pod
      │
      ▼
Service Name (student-db)
      │
      ▼
CoreDNS
      │
      ▼
Service IP
      │
      ▼
Service
      │
      ▼
Database Pod
```

### **Key Idea**

```text
CoreDNS:
"What IP belongs to student-db?"

Service:
"Which Pod should receive traffic sent to that IP?"
```

# **Kubernetes Service vs CoreDNS**

## **Common Misconception**

Many people think:

```text
Application Pod
      │
      ▼
student-db
      │
      ▼
Database Pod
```

and assume the Service name itself routes traffic.

This is not what happens.

The Service name must first be translated into an IP address.

That translation is performed by CoreDNS.

---

# **What Problem Does a Service Solve?**

Pods are ephemeral.

Example:

```text
Database Pod
IP: 10.244.1.5
```

Pod restarts:

```text
Database Pod
IP: 10.244.2.8
```

The IP changed.

If applications connect directly to Pod IPs, communication breaks.

A Kubernetes Service provides:

- Stable virtual IP (ClusterIP)
- Stable endpoint
- Load balancing
- Pod discovery using labels/selectors

Example:

```text
Service:
student-db

ClusterIP:
10.96.0.15

Selector:
app=database
```

Service always keeps track of matching Pods.

Even if Pod IPs change, the Service continues working.

---

# **What Problem Does CoreDNS Solve?**

Applications do not want to use:

```text
10.96.0.15
```

They want to use:

```text
student-db
```

or

```text
student-db.student-api.svc.cluster.local
```

Computers cannot route traffic using names.

Networking requires IP addresses.

CoreDNS provides DNS resolution.

Example:

```text
student-db
      ↓
10.96.0.15
```

This is the same concept as:

```text
google.com
      ↓
142.250.x.x
```

on the internet.

---

# **Responsibilities**

## **Service**

Service answers:

“Which Pod should receive this traffic?”

Example:

```text
10.96.0.15
      ↓
Database Pods
```

Responsibilities:

- Stable endpoint
- Load balancing
- Pod selection using labels
- Abstracting Pod IP changes

---

## **CoreDNS**

CoreDNS answers:

“What IP address belongs to this Service?”

Example:

```text
student-db
      ↓
10.96.0.15
```

Responsibilities:

- Service discovery
- DNS resolution
- Name → IP mapping

---

# **Real Request Flow**

Application configuration:

```env
DB_HOST=student-db
```

Application tries:

```text
http://student-db:5432
```

Actual flow:

```text
Application Pod
      │
      ▼
CoreDNS
      │
      ▼
student-db
      ↓
10.96.0.15
      │
      ▼
Service
      │
      ▼
Endpoint List
      │
      ▼
Database Pod
```

### **Detailed Flow**

Step 1:

Application asks:

```text
Who is student-db?
```

Step 2:

CoreDNS replies:

```text
student-db
      ↓
10.96.0.15
```

Step 3:

Application sends traffic to:

```text
10.96.0.15
```

Step 4:

Service receives traffic.

Step 5:

Service selects an endpoint:

```text
10.244.1.5
or
10.244.1.6
```

Step 6:

Database Pod receives request.

---

# **What Happens Without CoreDNS?**

CoreDNS stopped.

Application:

```bash
curl http://student-db
```

Result:

```text
Fails
```

Reason:

Application cannot resolve:

```text
student-db
```

to:

```text
10.96.0.15
```

The Service still exists.

The application simply cannot find it by name.

---

# **What Happens Without Service?**

CoreDNS asks Kubernetes:

```text
Where is student-db?
```

Kubernetes replies:

```text
No such Service
```

Resolution fails.

Even though CoreDNS is running.

---

# **Why Doesn’t Service Handle DNS?**

A Service is not a DNS server.

A Service only knows:

- ClusterIP
- Selectors
- Endpoints

Example:

```text
10.96.0.15
      ↓
Matching Pods
```

It does **not** answer:

```text
What IP belongs to student-db?
```

CoreDNS answers that question.

---

# **Fully Qualified Domain Name (FQDN)**

Kubernetes automatically creates DNS records.

Format:

```text
service.namespace.svc.cluster.local
```

Example:

```text
student-db.student-api.svc.cluster.local
```

Breakdown:

```text
student-db     → Service Name
student-api    → Namespace
svc            → Service Resource
cluster.local  → Cluster Domain
```

---

# **Service + CoreDNS Relationship**

Many engineers think:

```text
Service Name
      │
      ▼
Pod
```

Actual flow:

```text
Service Name
      │
      ▼
CoreDNS
      │
      ▼
Service IP
      │
      ▼
Service
      │
      ▼
Pod Endpoint
      │
      ▼
Pod
```

---

# **Analogy**

Imagine a restaurant.

Restaurant Name:

```text
McDonald's
```

Address:

```text
123 Main Street
```

### **CoreDNS**

Acts like Google Maps.

```text
McDonald's
      ↓
123 Main Street
```

### **Service**

Acts like the actual restaurant building.

```text
123 Main Street
      ↓
Customers served
```

CoreDNS helps you find the address.

Service handles traffic arriving at that address.

---

# **Hands-On Lab (Minikube)**

## **Step 1: Verify CoreDNS**

```bash
kubectl get pods -n kube-system
```

Expected:

```text
coredns-xxxxx
```

---

## **Step 2: Verify kube-dns Service**

```bash
kubectl get svc -n kube-system
```

Expected:

```text
NAME       TYPE        CLUSTER-IP
kube-dns   ClusterIP   10.96.0.10
```

Notice:

```text
CoreDNS itself is exposed through a Service.
```

---

## **Step 3: Create Test Namespace**

```bash
kubectl create ns dns-demo
```

---

## **Step 4: Deploy Nginx**

```bash
kubectl create deployment nginx \
  --image=nginx \
  -n dns-demo
```

---

## **Step 5: Create Service**

```bash
kubectl expose deployment nginx \
  --port=80 \
  --name=nginx-service \
  -n dns-demo
```

Verify:

```bash
kubectl get svc -n dns-demo
```

Example:

```text
NAME            CLUSTER-IP
nginx-service   10.98.123.45
```

---

## **Step 6: Launch Test Pod**

```bash
kubectl run dns-test \
  --image=busybox:1.36 \
  -it --rm \
  --restart=Never \
  -n dns-demo \
  -- sh
```

---

## **Step 7: Inspect DNS Configuration**

Inside the pod:

```bash
cat /etc/resolv.conf
```

Example:

```text
nameserver 10.96.0.10
search dns-demo.svc.cluster.local svc.cluster.local cluster.local
```

This proves:

```text
Pod
  ↓
CoreDNS (10.96.0.10)
```

---

## **Step 8: Resolve Service Name**

Inside the pod:

```bash
nslookup nginx-service
```

Expected:

```text
Name: nginx-service
Address: 10.98.123.45
```

This demonstrates:

```text
CoreDNS
     ↓
Service IP
```

---

## **Step 9: Access Service**

```bash
wget -qO- http://nginx-service
```

Expected:

```html
Welcome to nginx!
```

This demonstrates:

```text
CoreDNS
     ↓
Service
     ↓
Pod
```

---

## **Step 10: Observe the Complete Chain**

```text
wget http://nginx-service

      │
      ▼
CoreDNS

nginx-service
      ↓
10.98.123.45

      │
      ▼
Service

10.98.123.45
      ↓
nginx Pod

      │
      ▼
Response Returned
```

---

# **Key Takeaways**

- Pods have dynamic IP addresses.
- Services provide stable virtual IPs and load balancing.
- CoreDNS provides name resolution for Services.
- Services and CoreDNS solve different problems.
- Applications usually communicate using Service names, not Pod IPs.
- CoreDNS translates Service names into Service IPs.
- Services forward traffic from Service IPs to Pods.
- Both Service and CoreDNS are required for service-to-service communication in Kubernetes.

## **One-Line Summary**

```text
CoreDNS answers:
"What IP belongs to student-db?"

Service answers:
"Which Pod should receive traffic sent to that IP?"
```