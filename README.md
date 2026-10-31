# Kubernetes Microservices Production

A Kubernetes deployment of an online shop’s microservices on Linode Kubernetes Engine (LKE), with production and security best practices.

## Technologies Used

Kubernetes, Redis, Linux, Linode LKE

## Architecture Overview

The system is split into independent microservices that talk to each other over gRPC (the frontend serves HTTP to browsers). Only the `frontend` is reachable from outside the cluster; everything else uses internal `ClusterIP` services.

![Microservices connection map](images/microservices_full_connection_map.svg)

### Services

| Service | Port |
|---|---|
| frontend | 80 (LoadBalancer) -> 8080 |
| productcatalogservice | 3550 |
| currencyservice | 7000 |
| cartservice | 7070 |
| recommendationservice | 8080 |
| shippingservice | 50051 |
| paymentservice | 50051 |
| emailservice | 5000 -> 8080 |
| checkoutservice | 5050 |
| adservice | 9555 |
| redis-cart | 6379 |


## Repository Contents

```
.
├── config.yaml   # Deployments + Services for all microservices
└── README.md
```

## Prerequisites & Setup

### 1. Create the Cluster

In Linode, create an LKE cluster named `online-shop-microservices` in the region closest to you with 3 worker nodes (2 GB each). Install `kubectl` locally.

### 2. Cluster Connection & Security

Download the cluster configuration file (`online-shop-microservices-kubeconfig.yaml`). It contains your cluster credentials, so store it securely and never commit it to Git. Restrict permissions before exporting the path:

```bash
chmod 600 online-shop-microservices-kubeconfig.yaml
export KUBECONFIG=/path/to/online-shop-microservices-kubeconfig.yaml
```

Verify the connection:

```bash
kubectl get nodes
```

### 3. Namespace Creation

Create a dedicated namespace to isolate workloads and manage permissions:

```bash
kubectl create ns microservices
```

### 4. Deploying the Application

Apply the unified configuration manifest:

```bash
kubectl apply -f microservices-config.yaml -n microservices
```

Check running pods and services:

```bash
kubectl get pods -n microservices
kubectl get svc -n microservices
```

### 5. Access the Shop

Wait until the `frontend` service shows an `EXTERNAL-IP`, then open `http://<EXTERNAL-IP>` in your browser.

## Production & Security Best Practices

These are the practices to aims for. 

### Image Versioning

- **Pinned tags** specify explicit image versions (e.g. `:v0.10.7`) instead of `latest`, so deployments are reproducible and the running version is unambiguous.

### Reliability & Health Probes

- **Liveness probes** check periodically whether a container is healthy; when a probe fails, Kubernetes restarts the container. `initialDelaySeconds` gives the app time to start before the first check.
- **Readiness probes** determine when a container is ready to accept traffic. Without them, Kubernetes assumes a pod is ready as soon as the container starts.
- **Probe types** are `grpc` for gRPC services, `httpGet` (`/_healthz`) for the frontend, and `tcpSocket` for Redis.

### Resource Management

- **Requests and limits** serve two roles: requests are the guaranteed resources used for scheduling, and limits are the ceiling that prevents noisy-neighbor problems. CPU requests stay at 1 core or below (1 CPU = 1000m), and Redis receives more memory and less CPU because it is memory-heavy.
- **Node constraints** require that requests fit within the allocatable capacity of at least one worker node (which is lower than the node's advertised size, since the system reserves some); otherwise the pod can never be scheduled. Limits should also stay within what a node can realistically provide.

### Networking & Security

- **LoadBalancer and ClusterIP** provide a single external entry point through the cloud `LoadBalancer` (frontend), while internal services use `ClusterIP`. NodePort is avoided because it opens a port on every worker node.
- **Non-root execution** keeps containers from running as root, which limits the damage if one is compromised; official images are preferred, and their runtime user should be verified.
- **Image scanning** checks third-party and base images for vulnerabilities, both manually and automatically in CI/CD pipelines.
- **Namespaces and labels** isolate workloads and allow privileges to be set per namespace (via RBAC), while labels make resources selectable and organized.
- **Kubernetes upgrades** keep the cluster on a supported, recent version for security and bug fixes; nodes are upgraded one at a time so replicas keep the app available.

### Replicas & High Availability

- **Multiple replicas** prevent downtime when a pod crashes; the default of 1 replica would mean an outage, so most services run 2.
- **Multiple worker nodes** prevent a single node's crash, reboot, or maintenance from taking everything down. Replicas should run on different nodes so pods can be rescheduled to healthy ones.


## Clean Up

```bash
kubectl delete namespace microservices
```

Then delete the LKE cluster in Linode to stop billing.