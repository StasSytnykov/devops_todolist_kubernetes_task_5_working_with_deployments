# Deploying the ToDo app to Kubernetes

This document describes how to deploy the ToDo app to a Kubernetes cluster, and explains the choices made in the Deployment and HPA manifests.

## Prerequisites

- A running Kubernetes cluster (e.g. minikube, kind, or a cloud provider)
- `kubectl` configured to use that cluster

## How to deploy

1. **Create the namespace** (if it does not exist):

   ```bash
   kubectl create namespace mateapp
   ```

2. **Apply the Deployment**:

   ```bash
   kubectl apply -f .infrastructure/deployment.yml
   ```

3. **Apply the Horizontal Pod Autoscaler**:

   ```bash
   kubectl apply -f .infrastructure/hpa.yml
   ```

4. **Expose the app with a Service** (required to access it):

   ```bash
   kubectl apply -f .infrastructure/nodeport.yml
   ```

5. **Verify**:

   ```bash
   kubectl -n mateapp get deployment,pods,hpa,svc
   ```

   Wait until all pods are `Running` and `Ready`.

---

## Resource requests and limits

The Deployment defines the following for each container:

| Resource | Request | Limit   |
|----------|---------|---------|
| CPU      | 100m    | 500m    |
| Memory   | 128Mi   | 256Mi   |

**Why these values?**

- **Requests** are used by the scheduler to place pods and to guarantee that the node has enough capacity. They also define the basis for CPU/memory utilization used by the HPA.
  - **100m CPU**: Light API app; 0.1 core is enough to handle normal traffic and keeps scheduling flexible.
  - **128Mi memory**: Sufficient for a small Django app at idle and under light load.

- **Limits** cap maximum usage and prevent a single pod from consuming the whole node.
  - **500m CPU**: Allows short bursts (e.g. request spikes) without being too high for a typical node.
  - **256Mi memory**: Prevents memory growth (e.g. leaks or heavy usage) from affecting other workloads.

With two replicas and these requests, the cluster needs to fit 200m CPU and 256Mi memory for the app in idle state, which keeps the “2 pods running at idle” requirement realistic on small clusters.

---

## HPA configuration

The Horizontal Pod Autoscaler is configured as:

- **minReplicas: 2** — At least two pods always run for availability and to spread load.
- **maxReplicas: 5** — Caps scaling so the app doesn’t grow beyond a reasonable size for the task.
- **Metrics**: scale on both **CPU** and **memory**, each with **target average utilization: 70%**.

**Why scale on both CPU and memory?**

- CPU and memory don’t always correlate; one can be high while the other is low.
- Scaling only on CPU could leave memory pressure unsolved, and the opposite for memory-only scaling.
- Using both ensures extra pods are added when either CPU or memory usage is high (HPA scales when *any* metric is above the target).

**Why 70%?**

- Scaling at 70% gives headroom before pods hit limits and avoids constant scaling in/out around 100%.
- It keeps some capacity for traffic spikes and aligns with common practice for web/API workloads.

---

## Strategy configuration (RollingUpdate)

The Deployment uses a **RollingUpdate** strategy with:

- **maxUnavailable: 1**
- **maxSurge: 1**

**Why RollingUpdate?**

- New versions are rolled out by updating pods gradually instead of recreating all at once.
- There is no full downtime: old and new versions run in parallel during the rollout.

**Why maxUnavailable: 1 and maxSurge: 1?**

- **maxUnavailable: 1** — At most one existing pod can be unavailable during the update. With 2 replicas, at least one pod stays available so the app keeps serving traffic.
- **maxSurge: 1** — At most one extra pod can be created above the desired replica count. So during an update you have at most 3 pods (2 desired + 1 surge), which limits resource usage and keeps rollouts predictable on small clusters.

Together, this gives a safe, step-by-step rollout: one new pod is brought up, one old one is taken down, until all replicas are updated.

---

## How to access the app after deployment

After the Deployment, HPA, and Service are applied and pods are Ready:

1. **If you used a NodePort service** (e.g. `nodePort: 30080` in `mateapp`):
   - **Minikube**:  
     `http://$(minikube ip):30080`  
     Or: `minikube service todoapp -n mateapp`
   - **Kind / other clusters**:  
     Use any node’s IP and port **30080**:  
     `http://<NODE_IP>:30080`
   - API: `http://<NODE_IP>:30080/api/`  
   - Landing: `http://<NODE_IP>:30080/`

2. **If you use port-forward instead of NodePort**:

   ```bash
   kubectl -n mateapp port-forward svc/todoapp 8080:80
   ```

   Then open: `http://localhost:8080/` and `http://localhost:8080/api/`.

3. **From inside the cluster** (e.g. another pod in `mateapp`):  
   Use the ClusterIP service: `http://todoapp.mateapp.svc.cluster.local` (port 80).

Replace `<NODE_IP>` with the actual node IP from `kubectl get nodes -o wide` or your cluster’s access method.
