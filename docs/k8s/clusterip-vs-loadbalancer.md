# ClusterIP vs LoadBalancer: Internal Pod-to-Pod Communication

## Question

Let's say we have a Pod A which has 3 instances and then Pod B which relies on Pod A. In order to call Pod A from Pod B, do I need to make a LoadBalancer or can I just create a ClusterIP Service and the requests to Pod A will be load balanced by itself?

## Answer

**You do not need a LoadBalancer for Pod B to call Pod A inside the cluster.**

Create a **ClusterIP Service** for A, and B calls the Service name. Kubernetes will load-balance the requests across A's 3 pod instances automatically.

---

## Why This Works

- **ClusterIP is reachable from inside the cluster** (Pod → Service → Pods)
- The Service has a **stable DNS name** (e.g., `a-service`)
- The Service keeps an **up-to-date list of A's pod IPs** (endpoints)
- When B connects to `a-service`, **kube-proxy** (or eBPF) forwards the connection to one of A's pods

---

## The Usual Pattern

1. **Deployment A** (`replicas=3`) + **Service A** (`type: ClusterIP`)
2. **Deployment B** calls `http://a-service:<port>` (or `http://a-service.<namespace>:<port>`)

---

## Does It "Load Balance by Itself"?

**Yes**, with one important nuance:

- For HTTP (and most app traffic), clients typically open **connections**; the Service balances **connections** across pods
- If B uses **keep-alive / long-lived connections**, you might see "sticky-ish" behavior because:
  - Once a connection is established, it stays with that chosen pod until the connection is closed
  - New connections get balanced again

**So:** Balanced per connection, not per individual HTTP request in all cases.

---

## When Would You Use a LoadBalancer?

**Only if something outside the cluster** needs to reach A directly (or you're exposing an entrypoint like an Ingress controller). 

For **internal B→A calls**, ClusterIP is the standard.

---

## Summary

- ✅ **ClusterIP**: Use for internal pod-to-pod communication within the cluster
- ✅ **Automatic load balancing**: Kubernetes handles it via the Service
- ❌ **LoadBalancer**: Only needed for external access from outside the cluster
