# Kubernetes & DevOps — Week Three

A structured reference for Kubernetes architecture, control/data plane, Pods, ReplicaSet, Deployments, and debugging.

---

## Table of Contents

1. [Part 1: Kubernetes Overview](#part-1-kubernetes-overview)
2. [Part 2: Control Plane](#part-2-control-plane)
3. [Part 3: Data Plane (Worker Nodes)](#part-3-data-plane-worker-nodes)
4. [Part 4: Pods](#part-4-pods)
5. [Part 5: Pod Creation Flow](#part-5-pod-creation-flow)
6. [Part 6: ReplicaSet & Deployment](#part-6-replicaset--deployment)
7. [Part 7: Networking Recap](#part-7-networking-recap)
8. [Part 8: Debugging & Best Practices](#part-8-debugging--best-practices)

---

# Part 1: Kubernetes Overview

## What Kubernetes Is

- **Industry-standard container orchestrator** (open-source).
- Acts like a **“data center OS”**: automated deploy, scaling, self-healing, container lifecycle.
- **Declarative**: you specify desired state; K8s reconciles toward it.

## Architecture Model

| Plane | Role |
|-------|------|
| **Control Plane** | Decision-making and cluster management (API, scheduler, controllers, etcd) |
| **Data Plane** | Runs workloads (Pods) on worker nodes |

All cluster communication flows through the **API server**.

---

# Part 2: Control Plane

## Components

| Component | Role |
|-----------|------|
| **kube-apiserver** | Central gateway; every request goes through it. AuthN/AuthZ, REST API, stateless (can scale). Writes state to etcd. |
| **etcd** | Distributed key-value store: cluster config, desired state, metadata, secrets, Pod specs. **Losing etcd = losing the cluster** → backups essential. Data under `/registry/`. |
| **kube-scheduler** | Chooses which node runs each Pod. Uses: resources (CPU/mem), taints/tolerations, affinity, topology, priorities, node health. Sets `spec.nodeName`. |
| **kube-controller-manager** | Reconciliation loops: desired vs current state. Controllers: Node, Deployment, ReplicaSet, Job, ServiceAccount, Volume, etc. |
| **cloud-controller-manager** | Cloud integration (AWS, Azure, GCP): LBs, cloud volumes, routing, node lifecycle. |

---

# Part 3: Data Plane (Worker Nodes)

## Per-Node Components

| Component | Role |
|-----------|------|
| **kubelet** | Node agent. Talks to API server. Keeps containers running; health checks, restarts, volume mount, image pull. |
| **kube-proxy** | Network rules (iptables or IPVS). Routes traffic to Pod IPs; manages Service → Pod endpoints (east-west traffic). |
| **Container runtime** | Via CRI. Runs containers (containerd, CRI-O; Docker deprecated). Pulls images, creates containers, namespaces/cgroups. |
| **CNI plugin** | Pod networking: IP allocation, virtual interfaces. Examples: Calico, Cilium, Flannel, Weave. |
| **CSI plugin** | Persistent volumes: attach/mount. Backends: EBS, Azure Disk, GCP PD, NFS, Ceph. |

---

# Part 4: Pods

## What a Pod Is

- **Smallest deployable unit** in Kubernetes.
- One or more containers (usually one) that share:
  - Network namespace (one IP per Pod)
  - IPC namespace
  - Volumes
- **Ephemeral**: when a Pod is gone, its data is gone unless stored in volumes or external storage. Pods do not self-heal by themselves; controllers (e.g. ReplicaSet) recreate them.

## Pod in etcd

Path: `/registry/pods/<namespace>/<podName>`

| Section | Contents |
|--------|----------|
| `metadata` | uid, resourceVersion, creationTimestamp |
| `spec` | containers, volumes, nodeName, tolerations, affinity |
| `status` | phase, conditions, containerStatuses |

---

# Part 5: Pod Creation Flow

**End-to-end: `kubectl apply -f pod.yaml` → running Pod**

1. **Client** → HTTP POST to API server.
2. **API server** → AuthN (cert/token), AuthZ (RBAC). 401/403 if denied.
3. **Admission controllers** → Mutating (defaults, sidecars) and Validating (can reject).
4. **API server** → Writes Pod to etcd at `/registry/pods/`; gets `resourceVersion`.
5. **Scheduler** → Watches unscheduled Pods; filters (resources, taints, affinity), scores nodes; writes binding → sets `spec.nodeName`.
6. **Kubelet** (on chosen node) → Watches Pods with `spec.nodeName == self`; enqueues Pod.
7. **Kubelet** → Prepares volumes (CSI), networking (CNI → Pod IP), CRI `RunPodSandbox` (pause container), pulls images, creates/starts containers, runs init containers, updates `status` to API server.
8. **Probes** → Readiness: when OK, Pod is added to Service endpoints. Liveness: failure → container restart.

---

# Part 6: ReplicaSet & Deployment

## ReplicaSet

- Keeps a **fixed number of Pod replicas** running.
- Compares desired vs current; creates or deletes Pods to match.
- Selector: `matchLabels`; template: Pod spec.

**Flow:** kubectl apply → API server → etcd (`/registry/replicasets/`) → ReplicaSet controller sees RS → creates Pod objects (with `ownerReferences`) → Scheduler assigns nodes → Kubelets run containers. Loop: list Pods by selector → desired vs current → create/delete as needed.

**Failures:** Pod crash → RS creates new Pod. Node down → after ~5 min Pods Unknown → RS replaces. Manual delete → RS recreates.

## Deployment

- **Primary workload controller** for stateless apps.
- **Manages ReplicaSets** (and thus Pods).
- Adds: **rolling updates**, **rollback**, pause/resume, declarative updates.

ReplicaSet alone: no rolling update, no rollback. Deployment creates new ReplicaSet per template change and scales old one down.

### Deployment YAML (minimal)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: app
          image: nginx:1.27
```

### Rollout strategies

| Strategy | Behavior |
|----------|----------|
| **RollingUpdate** (default) | Replace Pods gradually. `maxUnavailable`, `maxSurge` (e.g. 25%). |
| **Recreate** | Terminate all old Pods, then create new. For apps that can’t run two versions at once. |

### Commands

| Task | Command |
|------|---------|
| Update image | `kubectl set image deployment/webapp app=nginx:1.28` |
| Rollout status | `kubectl rollout status deployment/webapp` |
| History | `kubectl rollout history deployment/webapp` |
| Rollback | `kubectl rollout undo deployment/webapp` |
| Pause / Resume | `kubectl rollout pause deployment/webapp` ; `kubectl rollout resume deployment/webapp` |
| Scale | `kubectl scale deployment webapp --replicas=10` |

### How updates work

New template → new template hash → **new ReplicaSet** created → old RS scaled down, new Pods created in new RS. Rollback = point Deployment at previous ReplicaSet revision.

**etcd:** `/registry/deployments/`, `/registry/replicasets/` (with ownerReference to Deployment), `/registry/pods/`.

---

# Part 7: Networking Recap

*(See WEEKTWO for full Docker networking.)*

- **IP, NIC, switch, ARP, gateway, NAT (SNAT/DNAT), subnetting** — same concepts underpin container networking.
- **Docker:** network namespace per container, veth pair, bridge (`docker0`), NAT for internet, port mapping `-p 8080:80`. Network types: bridge, host, none, overlay, macvlan. Custom bridge → DNS by container name.
- **Kubernetes:** CNI gives each Pod an IP; kube-proxy and Service objects route traffic to Pods. Network policies restrict traffic.

---

# Part 8: Debugging & Best Practices

## Debugging commands

| Task | Command |
|------|---------|
| Pods (all ns) | `kubectl get pods -A` |
| Pod details | `kubectl describe pod <pod> -n <ns>` |
| Pod YAML | `kubectl get pod <pod> -o yaml` |
| Logs | `kubectl logs <pod> -c <container>` |
| Events | `kubectl get events -n <ns> --sort-by='.lastTimestamp'` |
| Pending Pods | `kubectl get pods --field-selector=status.phase=Pending` |

## Common failures

| Symptom | Check |
|---------|--------|
| **Pending** | `kubectl describe pod` → scheduling: resources, nodeSelector, taints |
| **ImagePullBackOff** | Image name, pull secrets, registry auth |
| **CrashLoopBackOff** | `kubectl logs`, liveness probe config |
| **ContainerCreating (stuck)** | CNI (plugin logs), CSI (volume mount) |

## Architectural principles

- **Reconciliation:** Loop: read desired from etcd, read actual from cluster, if different → fix.
- **Declarative:** Specify *what*; K8s determines *how*; self-healing.
- **Optimistic concurrency:** `resourceVersion` on updates; 409 if stale.
- **Control vs data plane:** Control plane decides; data plane runs workloads; all via API server.

## Best practices

| Area | Practice |
|------|----------|
| **Workloads** | Use Deployments/StatefulSets, not bare Pods |
| **Resources** | Set requests and limits |
| **Probes** | Readiness (traffic only when ready), liveness (restart when unhealthy) |
| **Placement** | Affinity/anti-affinity, PodDisruptionBudget for availability |
| **etcd** | Back up regularly; monitor; secure communication |
| **Networking** | Custom networks, DNS, Network Policies where needed |

---

## Summary: Control vs Data Plane

```
┌─────────────────────────────────────────────────┐
│              CONTROL PLANE                      │
│  API Server ←→ etcd ←→ Scheduler / Controllers   │
└────────────────────────┬────────────────────────┘
                         │ REST API
                         ▼
┌────────────────────────────────────────────────┐
│              WORKER NODES                        │
│  kubelet | kube-proxy | Container Runtime       │
│  CNI (Pod IPs) | CSI (Volumes)                   │
│  ┌──────────────────────────────────────────┐   │
│  │  PODs  →  [Container] [Container] ...     │   │
│  └──────────────────────────────────────────┘   │
└────────────────────────────────────────────────┘
```

---

*End of Kubernetes notes — Week Three.*
