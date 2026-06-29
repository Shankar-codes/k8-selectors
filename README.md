# k8-selectors

A collection of Kubernetes manifest examples demonstrating **pod scheduling controls** — taints & tolerations, node affinity, node anti-affinity, pod affinity, pod anti-affinity, and a real-world Redis use case.

---

## Overview

Kubernetes gives you several mechanisms to control *where* pods are scheduled in your cluster. This repo walks through each of them in numbered order, from basic taints all the way to a production-style Redis + web-server deployment pattern.

| File | Concept |
|---|---|
| `01-taint.yaml` | Applying a taint to a node |
| `02-toleration.yaml` | Pod tolerating a node taint |
| `03-node-affinity.yaml` | Required + preferred node affinity rules |
| `04-node-anti-affinity.yaml` | Excluding pods from specific nodes/zones |
| `05-pod-affinity.yaml` | Co-locating pods in the same zone |
| `06-pod-anti-affinity.yaml` | Spreading pods away from each other |
| `07-redis-use-case.yaml` | Real-world Redis cache + web-server deployment |

---

## Concepts

### Taints & Tolerations

**Taints** are applied to nodes and repel pods from being scheduled on them unless the pod explicitly *tolerates* the taint.

```
kubectl taint nodes <node-name> project=roboshop:NoSchedule
```

`01-taint.yaml` shows a basic nginx Deployment. The comment at the bottom shows the `kubectl taint` command used to taint a specific EC2 node with `project=roboshop:NoSchedule`.

`02-toleration.yaml` shows a Deployment that pins to a specific node via `nodeSelector` and adds a toleration to allow scheduling on a node tainted with `project=ecommerce:NoSchedule`:

```yaml
tolerations:
  - key: "project"
    operator: "Equal"
    value: "ecommerce"
    effect: "NoSchedule"
```

---

### Node Affinity

Node affinity lets you constrain which nodes a pod can land on based on node labels — it's a more expressive replacement for `nodeSelector`.

`03-node-affinity.yaml` demonstrates both rule types:

- **`requiredDuringSchedulingIgnoredDuringExecution`** — Hard rule. Pod will only schedule on nodes in the specified availability zones (`us-east-1a` through `us-east-1d`) that also have `disktype=ssd`.
- **`preferredDuringSchedulingIgnoredDuringExecution`** — Soft rule. The scheduler prefers nodes labelled `project=roboshop` or `project=amazon`, but will still schedule elsewhere if none match.

---

### Node Anti-Affinity

Anti-affinity uses the `NotIn` operator to *exclude* certain nodes.

`04-node-anti-affinity.yaml` uses `requiredDuringSchedulingIgnoredDuringExecution` with `operator: NotIn` to ensure the pod never lands in `us-east-1a`:

```yaml
- key: topology.kubernetes.io/zone
  operator: NotIn
  values:
    - us-east-1a
```

---

### Pod Affinity

Pod affinity schedules a pod *near* other pods that match a label selector, scoped by a topology key (e.g. zone or hostname).

`05-pod-affinity.yaml` defines two pods:

- `pod-affinity-1` — a plain nginx pod labelled `app: pod-1`.
- `pod-affinity-2` — uses `podAffinity` to require it be scheduled in the **same availability zone** as any pod with label `app=pod-1`.

```yaml
podAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
          - key: app
            operator: In
            values:
              - pod-1
      topologyKey: topology.kubernetes.io/zone
```

---

### Pod Anti-Affinity

Pod anti-affinity schedules a pod *away from* other pods matching a label selector.

`06-pod-anti-affinity.yaml` defines `pod-affinity-3` which must be scheduled in a **different availability zone** than any pod labelled `app=pod-affinity-1` or `app=pod-affinity-2`.

```yaml
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
          - key: app
            operator: In
            values:
              - pod-affinity-1
              - pod-affinity-2
      topologyKey: topology.kubernetes.io/zone
```

---

### Real-World Use Case: Redis + Web Server

`07-redis-use-case.yaml` puts it all together with a realistic deployment pattern: a Redis cache tier and a web-server tier that need to be co-located per node, but spread across nodes for availability.

**Redis Deployment (`redis-cache`)**
- 3 replicas
- `podAntiAffinity` with `topologyKey: kubernetes.io/hostname` — ensures each Redis replica lands on a **different node**

**Web Server Deployment (`web-server`)**
- 3 replicas
- `podAntiAffinity` — ensures web-server pods spread across **different nodes**
- `podAffinity` — ensures each web-server pod lands on the **same node** as a Redis pod

The result: every node in the cluster gets exactly one Redis pod and one web-server pod — optimal for low-latency cache access with no single point of failure.

```
Node A: redis-cache-pod-1  +  web-server-pod-1
Node B: redis-cache-pod-2  +  web-server-pod-2
Node C: redis-cache-pod-3  +  web-server-pod-3
```

---

## Usage

Apply any manifest with `kubectl`:

```bash
kubectl apply -f 01-taint.yaml
kubectl apply -f 02-toleration.yaml
kubectl apply -f 03-node-affinity.yaml
kubectl apply -f 04-node-anti-affinity.yaml
kubectl apply -f 05-pod-affinity.yaml
kubectl apply -f 06-pod-anti-affinity.yaml
kubectl apply -f 07-redis-use-case.yaml
```

To taint a node before testing tolerations:

```bash
kubectl taint nodes <node-name> project=roboshop:NoSchedule
```

To remove the taint:

```bash
kubectl taint nodes <node-name> project=roboshop:NoSchedule-
```

---

## Prerequisites

- A running Kubernetes cluster (tested with EKS)
- `kubectl` configured to point at the cluster
- Nodes labelled with the appropriate topology and disk-type labels for affinity rules to match (e.g. `topology.kubernetes.io/zone`, `disktype=ssd`)

---

## Key Concepts Reference

| Term | Description |
|---|---|
| **Taint** | A node-level mark that repels pods |
| **Toleration** | A pod-level declaration that allows scheduling on tainted nodes |
| **Node Affinity** | Rules that attract pods to nodes matching label criteria |
| **Node Anti-Affinity** | Rules that repel pods from nodes matching label criteria (`NotIn` operator) |
| **Pod Affinity** | Rules that co-locate pods near other pods |
| **Pod Anti-Affinity** | Rules that spread pods away from other pods |
| **`requiredDuring...`** | Hard rule — pod won't schedule if not satisfied |
| **`preferredDuring...`** | Soft rule — scheduler tries its best but won't block |
| **topologyKey** | Defines the scope of affinity (e.g. zone, hostname) |
