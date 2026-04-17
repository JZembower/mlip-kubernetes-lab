# Lab 11: Teardown and Fresh Start Guide

How to completely tear down the current lab environment and start from scratch.

## Part 1: Tearing Down the Current Environment

Run these commands in order. If any resource doesn't exist, the command will simply report "not found" and you can move on.

### Step 1: Stop any running tunnels

If you have `minikube service flask-backend-service` running in a terminal, press `Ctrl+C` in that terminal to stop the tunnel.

### Step 2: Delete the backend Deployment and Service

```powershell
kubectl delete -f backend-deployment.yaml
```

This removes the `flask-backend-deployment` (and its Pods) and the `flask-backend-service`.

### Step 3: Delete the trainer CronJob and PVC

```powershell
kubectl delete -f trainer-deployment.yaml
```

This removes the `model-trainer-job` CronJob, any Jobs it spawned, and the `model-pvc` PersistentVolumeClaim (which holds the saved model file).

### Step 4: Verify everything is cleaned up

```powershell
kubectl get all
```

You should only see the default `kubernetes` ClusterIP service. No Pods, Deployments, CronJobs, or Jobs should remain.

```powershell
kubectl get pvc
```

Should return "No resources found" — the shared volume is gone.

### Step 5: (Optional) Stop Minikube entirely

```powershell
minikube stop
```

This shuts down the Minikube cluster but preserves its state. You can restart it later with `minikube start`.

To **completely delete** the cluster and start fully fresh:

```powershell
minikube delete
```

This removes the Minikube VM/container, all cluster data, and all cached images.

### Step 6: (Optional) Remove Docker images locally

```powershell
docker rmi jzembower10/model-trainer:1.0.0
docker rmi jzembower10/flask-backend:1.0.0
```

This frees up local disk space. The images still exist on Docker Hub and will be pulled again when needed.

---

## Part 2: Starting from Scratch

### Step 1: Refresh PowerShell PATH

If `minikube` or `kubectl` are not recognized, reload the PATH:

```powershell
$env:Path = [Environment]::GetEnvironmentVariable('Path', 'Machine') + ';' + [Environment]::GetEnvironmentVariable('Path', 'User')
```

### Step 2: Start Minikube

```powershell
minikube start
```

Wait for the "Done! kubectl is now configured" message.

### Step 3: Verify Minikube is running

```powershell
kubectl get po -A
```

You should see the system pods (coredns, etcd, kube-apiserver, etc.) all in `Running` status.

### Step 4: Log into Docker Hub

```powershell
docker login
```

Enter your Docker Hub username and password when prompted.

### Step 5: Build and push the trainer image

```powershell
docker build -t jzembower10/model-trainer:1.0.0 -f Dockerfile.trainer .
docker push jzembower10/model-trainer:1.0.0
```

### Step 6: Deploy the trainer CronJob

```powershell
kubectl apply -f trainer-deployment.yaml
```

### Step 7: Verify the CronJob is running

```powershell
kubectl get cronjobs
```

Wait about 1 minute, then:

```powershell
kubectl get jobs
```

Check the logs of a completed job:

```powershell
# Replace <job-name> with the name from kubectl get jobs
kubectl logs -f job/<job-name>
```

You should see "Starting model training" and "Model trained and saved".

### Step 8: Build and push the backend image

```powershell
docker build -t jzembower10/flask-backend:1.0.0 -f Dockerfile.backend .
docker push jzembower10/flask-backend:1.0.0
```

### Step 9: Deploy the backend

```powershell
kubectl apply -f backend-deployment.yaml
```

### Step 10: Open the service tunnel

In a dedicated terminal (keep it open):

```powershell
minikube service flask-backend-service
```

Note the URL it provides (e.g., `http://127.0.0.1:<port>`).

### Step 11: Verify load balancing

In a separate terminal, run this multiple times:

```powershell
Invoke-WebRequest -UseBasicParsing -Uri http://127.0.0.1:<port>/model-info | Select-Object -ExpandProperty Content
```

Replace `<port>` with the actual port from Step 10. The `"host"` field should alternate between two different pod names.

Test the predict endpoint:

```powershell
Invoke-WebRequest -UseBasicParsing -Uri http://127.0.0.1:<port>/predict -Method POST -ContentType "application/json" -Body '{"avg_session_duration":30,"visits_per_week":14,"response_rate":4,"feature_usage_depth":6,"user_id":34}' | Select-Object -ExpandProperty Content
```

### Step 12: Verify the preStop lifecycle hook

In one terminal, start streaming logs:

```powershell
kubectl logs -l app=flask-backend -f
```

In another terminal, trigger a rolling restart:

```powershell
kubectl rollout restart deployment/flask-backend-deployment
```

Watch the log output. You should see this sequence for each pod:

```
preStop signal received (SIGUSR1). Host preparing for shutdown: <pod-name>. ...
SIGTERM received. Host being terminated: <pod-name>. ...
```

The preStop message appears first, confirming the lifecycle hook runs before SIGTERM.

---

## Quick Reference: Full Teardown + Restart (Copy-Paste)

### Teardown

```powershell
kubectl delete -f backend-deployment.yaml
kubectl delete -f trainer-deployment.yaml
kubectl get all
minikube stop
```

### Restart

```powershell
minikube start
kubectl get po -A
docker build -t jzembower10/model-trainer:1.0.0 -f Dockerfile.trainer .
docker push jzembower10/model-trainer:1.0.0
kubectl apply -f trainer-deployment.yaml
docker build -t jzembower10/flask-backend:1.0.0 -f Dockerfile.backend .
docker push jzembower10/flask-backend:1.0.0
kubectl apply -f backend-deployment.yaml
minikube service flask-backend-service
```
