# Topology Spread Constraints and Multi-Zoned Nodes

## 1. Overview

**Topology Spread Constraints** control how Pods are distributed across your cluster's topology domains (zones, nodes, regions). They help ensure high availability by spreading workloads across failure domains.

---

## 2. What Problem Does This Solve?

### 2.1 Single Point of Failure

Without topology spread, all Pod replicas might end up on the same node or zone. If that node/zone fails, your entire service goes down.

### 2.2 Multi-Zone Clusters

Modern clusters often span multiple:
- **Availability zones** (different data centers in a region)
- **Regions** (different geographic locations)
- **Nodes** (physical/virtual machines)

You want Pods distributed across these domains for resilience.

---

## 3. Topology Spread Constraints

Topology spread constraints let you define:
- **How evenly** Pods should be distributed
- **Across which topology domains** (zones, nodes, etc.)
- **What happens** when domains are unbalanced

### 3.1 Basic Structure

```yaml
topologySpreadConstraints:
  - maxSkew: <integer>
    topologyKey: <string>
    whenUnsatisfiable: <string>
    labelSelector: <object>
```

**Key fields:**
- `maxSkew`: Maximum difference in Pod count between any two topology domains
- `topologyKey`: Node label key that defines topology domains (e.g., `topology.kubernetes.io/zone`)
- `whenUnsatisfiable`: `DoNotSchedule` (hard) or `ScheduleAnyway` (soft)
- `labelSelector`: Which Pods to count (usually matches your Deployment's Pod labels)

---

## 4. Common Topology Keys

### 4.1 Zone Distribution

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: web
```

**Result:** Pods spread evenly across zones. If you have 3 zones and 3 replicas, ideally 1 Pod per zone.

### 4.2 Node Distribution

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: web
```

**Result:** Pods spread evenly across nodes. Prevents all Pods from landing on a single node.

### 4.3 Combined Constraints

You can specify multiple constraints:

```yaml
topologySpreadConstraints:
  # Spread across zones
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: web
  # Also spread across nodes
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: web
```

---

## 5. Understanding MaxSkew

**MaxSkew** controls how "uneven" the distribution can be.

### 5.1 Example: MaxSkew = 1

- 3 zones, 3 replicas → ideally 1 Pod per zone
- If one zone has 2 Pods, another must have at least 1 Pod
- Maximum difference: 1 Pod

### 5.2 Example: MaxSkew = 2

- 3 zones, 5 replicas → distribution could be 2-2-1 or 2-1-2
- Maximum difference: 2 Pods between zones
- More flexible, allows some imbalance

**Rule:** Lower `maxSkew` = stricter distribution, higher = more flexible.

---

## 6. WhenUnsatisfiable

### 6.1 DoNotSchedule (Hard Constraint)

```yaml
whenUnsatisfiable: DoNotSchedule
```

- Pod **won't schedule** if constraint can't be satisfied
- Use when distribution is critical (e.g., high availability requirements)

### 6.2 ScheduleAnyway (Soft Constraint)

```yaml
whenUnsatisfiable: ScheduleAnyway
```

- Pod **will schedule** even if constraint can't be satisfied
- Scheduler tries to minimize skew but doesn't block scheduling
- Use when distribution is preferred but not critical

---

## 7. Multi-Zone Cluster Setup

### 7.1 Node Labels

Nodes in multi-zone clusters typically have labels:

```yaml
labels:
  topology.kubernetes.io/zone: us-east-1a
  topology.kubernetes.io/region: us-east-1
```

### 7.2 Zone-Aware Scheduling

With topology spread constraints, the scheduler:
1. Counts existing Pods per zone
2. Calculates skew between zones
3. Prefers zones with fewer Pods
4. Ensures `maxSkew` is not exceeded

---

## 8. Practical Example

### 8.1 Scenario

- **Cluster:** 3 zones (us-east-1a, us-east-1b, us-east-1c)
- **Deployment:** 6 replicas of a web app
- **Goal:** Even distribution across zones

### 8.2 Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 6
  template:
    metadata:
      labels:
        app: web
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: web
      containers:
        - name: app
          image: web-app:latest
```

### 8.3 Result

- Ideally: 2 Pods per zone
- If one zone fails: 4 Pods remain (2 in each surviving zone)
- High availability maintained

---

## 9. Topology Spread vs Pod Anti-Affinity

### 9.1 Similarities

Both help distribute Pods, but they work differently:

- **Pod Anti-Affinity:** "Don't put Pods together" (negative constraint)
- **Topology Spread:** "Spread Pods evenly" (positive constraint)

### 9.2 When to Use Which

**Use Pod Anti-Affinity when:**
- You want to prevent co-location
- Simple "don't schedule together" rule is enough

**Use Topology Spread when:**
- You want **even distribution** across domains
- You need fine-grained control over skew
- You want zone-aware or node-aware spreading

---

## 10. Common Patterns

### 10.1 High Availability Web Service

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: web
```

**Goal:** Ensure service survives zone failures.

### 10.2 StatefulSet with Zone Awareness

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app: database
```

**Goal:** Prefer zone distribution but don't block if impossible.

### 10.3 Node-Level Distribution

```yaml
topologySpreadConstraints:
  - maxSkew: 2
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: worker
```

**Goal:** Spread worker Pods across nodes, allow some imbalance.

---

## 11. Gotchas and Considerations

### 11.1 Label Selector Must Match Pod Labels

The `labelSelector` in topology spread constraints must match your Pod's labels. If it doesn't, the constraint won't work as expected.

### 11.2 Empty Topology Domains

If a topology domain has no matching Pods, it's considered to have 0 Pods. The scheduler will prefer placing Pods there to reduce skew.

### 11.3 Combined with Other Constraints

Topology spread works alongside:
- Node affinity/anti-affinity
- Resource requests/limits
- Taints and tolerations

The scheduler evaluates all constraints together.

### 11.4 Scaling Considerations

When scaling up:
- New Pods are placed to minimize skew
- Distribution improves as you add replicas

When scaling down:
- Pods are removed, but distribution may become uneven
- Consider using Pod Disruption Budgets (PDBs) to maintain availability

---

## 12. Summary

- **Topology Spread Constraints** ensure even distribution of Pods across topology domains (zones, nodes, regions)
- **MaxSkew** controls how uneven distribution can be
- **WhenUnsatisfiable** determines if constraint is hard (`DoNotSchedule`) or soft (`ScheduleAnyway`)
- **Multi-zone clusters** benefit from zone-aware topology spread for high availability
- Use **topology keys** like `topology.kubernetes.io/zone` or `kubernetes.io/hostname` to define domains
- More flexible than Pod Anti-Affinity for achieving even distribution

---

## 13. Quick Reference

**Common topology keys:**
- `topology.kubernetes.io/zone` - Availability zone
- `topology.kubernetes.io/region` - Region
- `kubernetes.io/hostname` - Node

**Best practices:**
- Use `maxSkew: 1` for critical services
- Use `DoNotSchedule` for high availability requirements
- Combine zone and node constraints for maximum resilience
- Test with different replica counts to ensure desired distribution
