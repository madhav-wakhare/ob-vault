# **iptables**

## **What is iptables?**

`iptables` is a Linux firewall and packet filtering framework used to control network traffic.

It defines rules that determine:

- Which packets are allowed
- Which packets are blocked
- Where packets should be forwarded

---

## **Why Kubernetes Uses iptables?**

Kubernetes Services have virtual IPs (ClusterIPs).

iptables creates rules that redirect traffic from a Service IP to the correct Pod IPs.

---

## **Flow**

```text
Request
   ↓
Service IP
   ↓
iptables Rules
   ↓
Pod IP
```

Example:

```text
10.96.0.10 (Service)
      ↓
iptables
      ↓
10.244.0.5 (Pod)
```

---

## **Kubernetes Components Using iptables**

- kube-proxy
- Service Load Balancing
- ClusterIP Routing
- NodePort Routing

---

## **Useful Commands**

```bash
sudo iptables -L
```

```bash
sudo iptables -t nat -L
```

```bash
sudo iptables-save
```

---

## **One-Line Summary**

```text
iptables is the Linux networking rule engine that Kubernetes uses to route Service traffic to the correct Pods.
```