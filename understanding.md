# Lab 11: Understanding Kubernetes for ML Training and Inference

## What This Lab Is About

This lab demonstrates how to use **Kubernetes** to orchestrate a machine learning pipeline that continuously trains a model and serves predictions through a load-balanced inference service. It covers three core Kubernetes concepts: CronJobs for scheduled work, Services for traffic distribution, and container lifecycle hooks for graceful shutdowns.

---

## Key Technologies

### Kubernetes

Kubernetes (K8s) is a container orchestration platform that automates the deployment, scaling, and management of containerized applications. Instead of manually starting and stopping containers on individual machines, you describe your desired state in YAML manifests, and Kubernetes makes it happen.

### Minikube

Minikube is a tool that runs a single-node Kubernetes cluster locally inside a Docker container (or VM). It provides a full Kubernetes environment for development and learning without needing cloud infrastructure.

### Docker

Docker packages applications and their dependencies into **images** that run identically anywhere. In this lab, we build two images:
- **model-trainer** — a Python script that trains a RandomForestRegressor and saves it to disk
- **flask-backend** — a Flask web server that loads the trained model and serves predictions

These images are pushed to **Docker Hub** (a public registry) so that Kubernetes can pull them when creating containers.

---

## Kubernetes Resources Used in This Lab

### PersistentVolumeClaim (PVC)

A PVC requests storage from the cluster. In this lab, `model-pvc` provides a 1Gi shared volume mounted at `/shared-volume` in both the trainer and backend containers. This is how the trainer saves the model file (`model.joblib`) and the backend reads it — they share the same persistent disk.

### CronJob

A CronJob runs a container on a schedule, similar to a Unix cron task. Our `model-trainer-job` uses the schedule `*/1 * * * *` (every minute) to retrain the model with fresh synthetic data. Each run creates a **Job**, which creates a **Pod**, which runs the training container to completion.

Key settings:
- `concurrencyPolicy: Forbid` — prevents a new training job from starting if the previous one is still running
- `ttlSecondsAfterFinished: 600` — automatically cleans up completed Jobs after 10 minutes
- `restartPolicy: OnFailure` — restarts the container if it crashes, but not if it succeeds

### Deployment

A Deployment manages a set of identical Pods (replicas). Our `flask-backend-deployment` maintains **2 replicas** of the Flask backend. If a Pod crashes, the Deployment controller automatically creates a replacement. During updates, it performs a **rolling update** — starting new Pods before terminating old ones to avoid downtime.

### Service

A Service provides a stable network endpoint for a set of Pods. Our `flask-backend-service` is a **NodePort** Service that:
- Assigns a stable internal ClusterIP that load-balances across all backend Pods
- Exposes the service externally on port `30080` on every node

### Pod

A Pod is the smallest deployable unit in Kubernetes. It runs one or more containers sharing the same network namespace and storage volumes. Each of our backend replicas is a Pod running the Flask container.

---

## How Load Balancing Works

When you send a request to the Service, Kubernetes distributes it to one of the backend Pods. Here's how:

1. The **Service** object has a virtual ClusterIP that doesn't correspond to any single Pod.
2. **kube-proxy**, running on every node, sets up **iptables rules** (or IPVS rules) that intercept traffic to the ClusterIP.
3. These rules randomly select one of the Pod IPs (Endpoints) backing the Service.
4. The request is forwarded to that Pod.

This is why the `"host"` field in our API responses alternates between different pod names — each request may land on a different replica. This distribution happens transparently with no application-level changes needed.

---

## How the preStop Lifecycle Hook Works

### The Problem

When Kubernetes needs to shut down a Pod (during a rolling update, scale-down, or node drain), it sends **SIGTERM** to the main process. But the application might be in the middle of handling a request, writing to disk, or holding a connection. An abrupt shutdown could cause errors.

### The Solution: preStop Hook

A **preStop hook** is a command that Kubernetes runs inside the container *before* sending SIGTERM. This gives the application a chance to prepare for shutdown.

In our lab, the preStop hook does two things:
```yaml
command: ["/bin/sh", "-c", "kill -10 1 && sleep 10"]
```

1. `kill -10 1` — sends signal 10 (SIGUSR1) to PID 1 (the Python/Flask process). The application has a signal handler that logs "preStop signal received" when it gets this signal.
2. `sleep 10` — pauses for 10 seconds, giving the application time to finish in-flight requests before Kubernetes sends SIGTERM.

### The Complete Shutdown Sequence

1. **Pod marked as Terminating** — Kubernetes removes the Pod from the Service's Endpoints list, so no new traffic is routed to it.
2. **preStop hook runs** — the `kill -10 1 && sleep 10` command executes. The application logs the preStop message.
3. **SIGTERM sent** — after the preStop hook completes, Kubernetes sends SIGTERM to the main process. The application logs the SIGTERM message and exits.
4. **Grace period** — the application has up to `terminationGracePeriodSeconds` (default 30s) to shut down.
5. **SIGKILL** — if the process is still running after the grace period, Kubernetes forcefully kills it.

### Practical Use Cases for Lifecycle Hooks

- **Draining in-flight requests** — stop accepting new requests and wait for current ones to finish
- **Deregistering from service discovery** — notify external load balancers or registries that this instance is going away
- **Flushing data** — write cached data, logs, or metrics to persistent storage before shutdown
- **Releasing resources** — close database connections, release locks, or clean up temporary files
- **Saving model state** — in ML systems, save the current model checkpoint or inference statistics

---

## How the ML Pipeline Fits Together

```
                    Shared Volume (PVC)
                    /shared-volume/model.joblib
                          |
         writes           |          reads
      +--------+          |       +----------+
      | Trainer |  -----> | <---- | Backend  |
      | CronJob |         |       | (Pod 1)  |
      +--------+          |       +----------+
      Runs every           |
      1 minute             |       +----------+
                           | <---- | Backend  |
                                   | (Pod 2)  |
                                   +----------+
                                        ^
                                        |
                                   +---------+
                                   | Service |  <--- User requests
                                   +---------+
                                   Load balances
                                   across replicas
```

1. The **CronJob** trains a new model every minute using synthetic data and saves it to the shared volume.
2. The **backend Pods** periodically reload the model from the shared volume (every 30 seconds via a background thread).
3. The **Service** distributes incoming `/model-info` and `/predict` requests across the backend replicas.
4. When Pods are restarted, the **preStop hook** ensures graceful shutdown before termination.

---

## Key Takeaways

- **Kubernetes automates** the deployment, scaling, and lifecycle management of containerized ML systems.
- **CronJobs** enable scheduled, repeatable tasks like periodic model retraining.
- **Services** provide built-in load balancing across multiple replicas with no application code changes.
- **Lifecycle hooks** give applications the ability to handle shutdowns gracefully, which is critical for production ML systems that need to finish serving predictions before going down.
- **PersistentVolumeClaims** allow containers to share data (like model files) across different workloads.
