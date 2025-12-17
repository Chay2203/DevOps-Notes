# Kubernetes Advanced & CI/CD — Week Four

A structured reference for HPA, Kubernetes networking, Services, and GitHub Actions.

---

## Table of Contents

1. [Part 1: Horizontal Pod Autoscaler (HPA)](#part-1-horizontal-pod-autoscaler-hpa)
2. [Part 2: Kubernetes Networking](#part-2-kubernetes-networking)
3. [Part 3: Kubernetes Services](#part-3-kubernetes-services)
4. [Part 4: GitHub Actions CI/CD](#part-4-github-actions-cicd)

---

# Part 1: Horizontal Pod Autoscaler (HPA)

## What HPA Does

HPA **scales a Deployment (or similar) up or down** based on metrics (e.g. CPU or memory). It changes replica count; it does not change resources per Pod.

**Typical setup:** CPU target (e.g. 50%) → HPA increases replicas when average Pod CPU &gt; target, decreases when below.

---

## Flow

```
Traffic → Service → Pods (Deployment)
                        ↑
Metrics Server → HPA Controller (evaluation loop) → scales replicas
```

---

## Step 1: Deploy Metrics Server

HPA needs **live CPU/memory** from the cluster. Without Metrics Server:

- HPA shows `0%/50%` (or &lt;unknown&gt;)
- Scaling does not happen

**Install (example):**

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

**Verify:**

```bash
kubectl get pods -n kube-system | grep metrics-server
kubectl top nodes
kubectl top pods
```

---

## Step 2: Deployment with CPU Requests

HPA uses:

```
utilization = (current CPU usage) / (CPU request)
```

If **`requests.cpu` is missing**, HPA cannot compute utilization and will not scale. Always set `requests.cpu` (and optionally `limits.cpu`) on containers that HPA targets.

**Minimal deployment (for HPA demo):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-demo-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      run: hpa-demo-deployment
  template:
    metadata:
      labels:
        run: hpa-demo-deployment
    spec:
      containers:
        - name: app
          image: registry.k8s.io/hpa-example
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 200m
            limits:
              cpu: 500m
```

```bash
kubectl apply -f deployment.yml
```

---

## Step 3: Service

Exposes Pods and gives a stable name for generating load.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hpa-demo-deployment
spec:
  selector:
    run: hpa-demo-deployment
  ports:
    - port: 80
```

```bash
kubectl apply -f service.yaml
```

---

## Step 4: Create HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-demo-deployment
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-demo-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

```bash
kubectl apply -f hpa.yaml
kubectl get hpa -w
```

Initially you may see `0%/50%` until Metrics Server reports. Then load will drive scaling.

---

## Step 5: Generate Load

From inside the cluster (e.g. one-off Pod):

```bash
kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- \
  /bin/sh -c "while sleep 0.01; do wget -q -O- http://hpa-demo-deployment; done"
```

Watch replicas and HPA:

```bash
kubectl get hpa -w
kubectl get deployment hpa-demo-deployment -w
kubectl top pods
```

Stop load (Ctrl+C); after cooldown, HPA scales down.

---

## Common Issues

| Symptom                         | Cause                                  | Fix                                                         |
| ------------------------------- | -------------------------------------- | ----------------------------------------------------------- |
| CPU shows 0% or &lt;unknown&gt; | Metrics Server not running / not ready | Install and verify Metrics Server; check `kubectl top pods` |
| No scaling                      | No `requests.cpu` on containers        | Set `resources.requests.cpu` in Pod template                |
| HPA not affecting Deployment    | Wrong `scaleTargetRef` or name         | Match `kind` and `name` to the Deployment                   |

---

## Takeaways

- **Metrics Server** is required for CPU/memory-based HPA.
- **CPU requests** are required for CPU utilization target.
- HPA reacts to **average** utilization across Pods of the target.
- Scaling is gradual (scale up and down with cooldown). For custom or external metrics, use **custom metrics API** or **KEDA**.

---

# Part 2: Kubernetes Networking

## Design Principles (3 Rules)

1. **All Pods** can talk to all other Pods **without NAT**.
2. **All Nodes** can talk to all Pods **without NAT**.
3. A **Pod’s IP** is the same from inside and outside the Pod.

So the model must support: **container-to-container**, **pod-to-pod**, **pod-to-service**, **internet-to-service**.

---

## Container-to-Container (Inside a Pod)

Containers in the same Pod **share the network namespace**: one IP, shared ports, shared loopback. They reach each other via **localhost**.

---

## Pod-to-Pod

Each Pod has its own network namespace. CNI assigns an IP and connects the Pod to the node network.

- **Same node:** veth pair → bridge (e.g. `cbr0`) → veth to other Pod. Traffic stays on the node.
- **Different nodes:** Pod CIDR per node (e.g. Node1 `10.0.1.0/24`, Node2 `10.0.2.0/24`). Traffic goes: Pod → bridge → node eth0 → network → destination node → bridge → destination Pod. Routing is CNI-specific (overlays, BGP, or cloud CNI like VPC CNI).

---

## Pod-to-Service (ClusterIP)

Pods are ephemeral; IPs change. A **Service** gives a **stable virtual IP (ClusterIP)** and DNS name, and load-balances to Pod endpoints.

- **kube-proxy** implements this with **iptables** or **IPVS** (DNAT from ClusterIP to a backend Pod).
- **DNS:** CoreDNS creates records like `myservice.namespace.svc.cluster.local`.

---

## Internet-to-Service

| Direction                     | Mechanism                                                                                                                                                         |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Egress (Pod → Internet)**   | Node does SNAT (Pod IP → Node IP); cloud gateway then NATs to public IP.                                                                                          |
| **Ingress (Internet → Pods)** | **LoadBalancer** Service (cloud LB → NodePort → Service → Pods) or **Ingress** (L7: host/path routing, TLS) via an Ingress controller (e.g. NGINX, ALB, Traefik). |

---

## Summary Table

| Layer                | What happens                           |
| -------------------- | -------------------------------------- |
| Container–Container  | Same network namespace → localhost     |
| Pod–Pod (same node)  | veth → bridge → veth                   |
| Pod–Pod (cross node) | Routed by Pod CIDR / CNI               |
| Pod–Service          | kube-proxy (iptables/IPVS) DNAT to Pod |
| DNS                  | CoreDNS: `svc.ns.svc.cluster.local`    |
| Egress               | SNAT Pod → Node → Internet             |
| Ingress              | LoadBalancer or Ingress controller     |

---

# Part 3: Kubernetes Services

## Why Services

Pods get new IPs when recreated; replicas have different IPs. A **Service** provides:

- Stable **ClusterIP** (and optional NodePort / LoadBalancer)
- Stable **DNS** name
- **Load balancing** across ready Pod endpoints (via kube-proxy)

---

## Service Types

| Type                    | Purpose                      | Access                                 |
| ----------------------- | ---------------------------- | -------------------------------------- |
| **ClusterIP** (default) | Internal only                | Inside cluster (DNS or ClusterIP:port) |
| **NodePort**            | Expose on each node          | `NodeIP:NodePort` (30000–32767)        |
| **LoadBalancer**        | Cloud LB in front of Service | External IP from cloud provider        |

---

## ClusterIP

Internal communication (e.g. frontend → backend, backend → DB).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
```

Access: `http://backend-svc` or `http://backend-svc.default.svc.cluster.local`.

---

## NodePort

Exposes the Service on every node at a high port.

```yaml
spec:
  type: NodePort
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 32080
```

Access: `http://<NodeIP>:32080`. Good for dev/bare-metal; not ideal for production internet.

---

## LoadBalancer

Cloud provider provisions a load balancer; it targets NodePort (or equivalent). Use for **public** or **VPC** access.

```yaml
spec:
  type: LoadBalancer
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
```

`kubectl get svc` shows **EXTERNAL-IP**. Access: `http://<EXTERNAL-IP>`.

---

## Other Concepts

| Concept                          | Use                                                                 |
| -------------------------------- | ------------------------------------------------------------------- |
| **Headless** (`clusterIP: None`) | DNS returns Pod IPs; used with StatefulSets, DBs.                   |
| **Multi-port**                   | Multiple entries in `spec.ports` with `name`, `port`, `targetPort`. |
| **ExternalName**                 | CNAME to external hostname (e.g. `database.company.com`).           |
| **Service without selector**     | You manage Endpoints manually; useful for external backends.        |

**DNS:** `<service>.<namespace>.svc.cluster.local`.

---

## Which Type to Use

| Need                            | Service type              |
| ------------------------------- | ------------------------- |
| Internal microservices          | ClusterIP                 |
| Expose on node (dev/bare-metal) | NodePort                  |
| Production external access      | LoadBalancer (or Ingress) |
| StatefulSet / per-Pod DNS       | Headless                  |
| Point to external DNS name      | ExternalName              |

---

# Part 4: GitHub Actions CI/CD

## What GitHub Actions Is

Automation that runs **workflows** (jobs and steps) on **events** (e.g. push, pull_request). Workflows are YAML files in `.github/workflows/`.

---

## Concepts

| Concept      | Meaning                                                        |
| ------------ | -------------------------------------------------------------- |
| **Workflow** | One YAML file; defines when and what runs.                     |
| **Job**      | Set of steps; runs on one runner.                              |
| **Step**     | Single action or script.                                       |
| **Runner**   | VM (e.g. Ubuntu) that runs the job.                            |
| **Action**   | Reusable unit (e.g. `actions/checkout`, `actions/setup-java`). |

---

## Example: Java Build on Push to main

**Layout:**

```
repo/
├── src/.../Main.java
├── pom.xml
└── .github/workflows/build.yml
```

**build.yml:**

```yaml
name: Java Build Workflow

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: "17"
          distribution: "temurin"

      - name: Build with Maven
        run: mvn clean package
```

---

## What Happens on Push to main

1. GitHub triggers the workflow.
2. A runner (e.g. Ubuntu) is allocated.
3. Steps run: checkout → install Java → `mvn clean package`.
4. Success → green check; failure → red X and logs in the Actions tab.

---

## Where to See Results

**Repo → Actions** → select run → open the **build** job and each step for logs.

---

## Common Mistakes

| Mistake               | Fix                                                                    |
| --------------------- | ---------------------------------------------------------------------- |
| Java version mismatch | Use same version in `pom.xml` and `setup-java`                         |
| Workflow not running  | Ensure `.github/workflows/` and filename/ext (e.g. `.yml`) are correct |
| YAML errors           | Use spaces (e.g. 2) not tabs; check indentation                        |
| Maven not found       | Add `actions/setup-java` before `mvn`                                  |

---

_End of Week Four notes._
