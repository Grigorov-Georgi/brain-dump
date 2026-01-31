# Kubernetes Building Blocks

This document explains the core Kubernetes (K8s) building blocks—**cluster**, **node**, **namespace**, **pod**, and **containers in pods**—and how they relate to each other.

---

## Cluster

A **Kubernetes cluster** is the whole system that runs your applications.

It includes:

- **Control plane** components (the "brains" that decide what should run where)
- **Worker nodes** (the machines that actually run your workloads)

**Mental model:**  
A cluster is like a "data center abstraction." You tell the cluster what you want (desired state), and it works to make it happen.

---

## Node

A **node** is a single machine (physical or virtual) that runs workloads.

A node typically provides:

- CPU, memory, networking
- A container runtime (e.g., containerd)
- A Kubernetes agent (`kubelet`) that makes sure the right pods/containers run on the node

**Key point:**  
Pods are scheduled onto **nodes**. Nodes are the "execution environment" for pods.

---

## Namespace

A **namespace** is a logical partition inside a cluster used to organize and isolate resources.

Namespaces help with:

- **Organization** (team-a vs team-b, dev vs prod)
- **Access control** (RBAC can grant permissions per namespace)
- **Resource management** (quotas/limits per namespace)
- **Name scoping** (many resource names must be unique *only within* a namespace)

**Key point:**  
Most Kubernetes resources (like Pods, Services, Deployments) live **in a namespace**.  
Nodes do **not** belong to a namespace (nodes are cluster-scoped).

---

## Pod

A **pod** is the smallest deployable unit in Kubernetes.  
It represents one or more containers that must run together.

A pod provides:

- A shared **network identity** (one IP address per pod)
- Shared **ports** (containers coordinate via `localhost`)
- Shared **storage volumes** (if configured)
- A single scheduling unit (the scheduler places the whole pod on one node)

**Key point:**  
Kubernetes schedules **pods**, not individual containers.

---

## Containers in Pods

A **container** is a packaged application process (image + runtime environment).  
In Kubernetes, containers are almost always run **inside pods**.

### Why multiple containers in a pod?

- **Sidecar pattern:** add a helper container (logging agent, proxy, etc.)
- **Tight coupling:** processes must share network/storage and be deployed together

### What containers in the same pod share:

- **Network namespace** → same pod IP, talk via `localhost`
- **Volumes** (if mounted to both containers)

### What they do *not* automatically share:

- Process space (unless special configuration like `shareProcessNamespace` is used)
- File system (other than shared volumes)
- Resource limits (each container can have its own requests/limits)

---

## How They Connect

### Big Picture Hierarchy

- A **cluster** contains:
  - **nodes** (machines)
  - **namespaces** (logical partitions)
- A **namespace** contains:
  - **pods** (and many other resources)
- A **pod** contains:
  - **one or more containers**
- A **pod** runs on:
  - **exactly one node** at a time (it can be rescheduled to another node if recreated)

### Important "Scope" Rules

- **Cluster-scoped:** Nodes (and some resources like PersistentVolumes, ClusterRoles)
- **Namespace-scoped:** Pods, Services, Deployments, ConfigMaps, Secrets, etc.

So:

- You choose a namespace to organize workloads.
- Kubernetes schedules your pods onto nodes.
- Each pod runs one or more containers.

---

## Concrete Example

Imagine this setup:

- **Cluster:** `prod-cluster`
- **Namespaces:** `payments`, `search`
- **Nodes:** `node-a`, `node-b`, `node-c`

In namespace `payments` you might have a pod:

- **Pod:** `checkout-api-7f9c...`
  - **Container 1:** `app` (your API server)
  - **Container 2:** `proxy` (a sidecar proxy for mTLS/traffic)

Kubernetes schedules that pod onto (say) `node-b`.  
Both containers run on `node-b`, inside the same pod network identity.

---

## Quick Mental Model

- **Cluster:** the whole Kubernetes environment
- **Node:** a machine where workloads run
- **Namespace:** a folder-like partition to group/isolate resources
- **Pod:** the smallest scheduled unit; a wrapper around one or more containers
- **Container:** the actual running app process(es), packaged as images, living inside pods

---

## Summary Diagram

```
Kubernetes Cluster
├── Nodes (machines)
│   ├── Node A
│   └── Node B
└── Namespaces (logical partitions)
    ├── Namespace: payments
    │   └── Pod: checkout-api
    │       ├── Container: app
    │       └── Container: sidecar-proxy
    └── Namespace: search
        └── Pod: query-api
            └── Container: app
```

---

## Common Misconceptions

1. **"A namespace is a node group."**  
   No—namespaces are logical; nodes are physical/virtual machines.

2. **"Containers are scheduled individually."**  
   No—**pods** are scheduled. Containers come along as part of the pod.

3. **"Pods are permanent."**  
   Pods are often ephemeral. Higher-level controllers (like Deployments) recreate them.

---

## Next Steps

If you want, I can extend this document with related resources (Deployment, ReplicaSet, Service, Ingress) so the "how apps are deployed and exposed" story is complete.
