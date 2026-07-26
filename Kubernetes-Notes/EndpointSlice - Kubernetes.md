# **EndpointSlice**

## **What is EndpointSlice?**

An **EndpointSlice** is a Kubernetes resource that stores the list of Pod endpoints behind a Service.

It replaces the older `Endpoints` resource and scales better for large clusters.

---

## **Why is it Needed?**

When a Service receives traffic, Kubernetes needs to know:

```text
Which Pods should receive this request?
```

EndpointSlice provides that information.

---

## **Flow**

```text
Service
   ↓
EndpointSlice
   ↓
Pod IPs
   ↓
Pods
```

Example:

```text
student-api-service
        ↓
EndpointSlice
        ↓
10.244.0.5
10.244.0.6
10.244.0.7
```

---

## **Benefits**

- Better scalability
- Supports thousands of endpoints
- Faster updates
- Lower API Server load

---

## **Useful Commands**

```bash
kubectl get endpointslices
```

```bash
kubectl get endpointslices -A
```

```bash
kubectl describe endpointslice <name>
```

---

## **One-Line Summary**

```text
Service provides a stable IP.
EndpointSlice stores the Pod IPs behind that Service.
```