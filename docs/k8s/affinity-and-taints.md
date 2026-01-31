# Affinity and Node Taints

## 1. Overview

There are mechanisms to control **where Pods are scheduled**: affinity rules influence placement based on labels, while taints prevent Pods from being scheduled on specific nodes.

---

## 2. Pod Affinity & Anti-Affinity

### 2.1 Pod Affinity (Co-location)

**Pod affinity** schedules Pods **together** based on labels.

**Use case:** Place related Pods on the same node/zone for performance (e.g., web server + cache).

```yaml
affinity:
  podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: cache
        topologyKey: kubernetes.io/hostname
```

### 2.2 Pod Anti-Affinity (Separation)

**Pod anti-affinity** schedules Pods **apart** to avoid single points of failure.

**Use case:** Spread replicas across nodes/zones for high availability.

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: web
          topologyKey: kubernetes.io/hostname
```

**Key difference:**
- `requiredDuringScheduling`: Hard requirement (Pod won't schedule if condition can't be met)
- `preferredDuringScheduling`: Soft preference (best effort)

---

## 3. Node Affinity

**Node affinity** schedules Pods on nodes with specific labels.

**Use case:** Run GPU workloads on GPU nodes, or place Pods in specific zones.

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: node-type
              operator: In
              values:
                - gpu
```

---

## 4. Node Taints & Tolerations

### 4.1 Taints (Node-Level)

A **taint** marks a node as "unwanted" for most Pods.

**Use case:** Reserve nodes for specific workloads (e.g., dedicated database nodes).

```bash
kubectl taint nodes node1 dedicated=database:NoSchedule
```

**Taint effects:**
- `NoSchedule`: Don't schedule new Pods (unless they tolerate)
- `PreferNoSchedule`: Try to avoid scheduling
- `NoExecute`: Evict existing Pods that don't tolerate

### 4.2 Tolerations (Pod-Level)

A **toleration** allows a Pod to be scheduled on a tainted node.

```yaml
tolerations:
  - key: dedicated
    operator: Equal
    value: database
    effect: NoSchedule
```

---

## 5. Quick Mental Model

- **Affinity**: "I want to be with/away from Pods/nodes matching these labels"
- **Taints**: "Don't schedule Pods here unless they explicitly tolerate"
- **Tolerations**: "I'm allowed on tainted nodes"

---

## 6. Common Patterns

1. **High availability**: Use pod anti-affinity to spread replicas
2. **Dedicated nodes**: Taint nodes, add tolerations to privileged Pods
3. **Co-location**: Use pod affinity for related services
4. **Zone awareness**: Use topology keys (`topology.kubernetes.io/zone`) for multi-zone clusters

---

## 7. Summary

- **Affinity**: Influences Pod placement based on labels (can be required or preferred)
- **Anti-affinity**: Keeps Pods apart (critical for HA)
- **Taints**: Node-level "keep away" signal
- **Tolerations**: Pod-level "I'm allowed" permission
