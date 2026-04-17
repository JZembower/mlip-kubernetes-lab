# Lab 11 Commands Reference

A step-by-step record of every command run during this lab and why.

---

## Prerequisites: Setting Up the Environment

```powershell
# Refresh the PATH so PowerShell can find newly installed tools (minikube, kubectl)
$env:Path = [Environment]::GetEnvironmentVariable('Path', 'Machine') + ';' + [Environment]::GetEnvironmentVariable('Path', 'User')
```

```powershell
# Start the local Kubernetes cluster using Minikube with the Docker driver
minikube start
```

```powershell
# Verify all system pods are running in the cluster to confirm Minikube is healthy
kubectl get po -A
```

---

## Task 1: Continuous Model Training with CronJobs

### Build and push the trainer image

```powershell
# Build the Docker image for the model trainer using Dockerfile.trainer
# Tags it with our Docker Hub username so it can be pushed to the registry
docker build -t jzembower10/model-trainer:1.0.0 -f Dockerfile.trainer .
```

```powershell
# Push the trainer image to Docker Hub so Kubernetes can pull it
docker push jzembower10/model-trainer:1.0.0
```

### Deploy the CronJob

```powershell
# Apply the trainer manifest, which creates:
#   - A PersistentVolumeClaim (model-pvc) for shared model storage
#   - A CronJob (model-trainer-job) that trains the model every minute
kubectl apply -f trainer-deployment.yaml
```

### Verify the CronJob

```powershell
# List CronJobs to confirm model-trainer-job exists with the correct schedule
kubectl get cronjobs
```

```powershell
# List completed/running Jobs spawned by the CronJob
kubectl get jobs
```

```powershell
# View the logs from a specific training job to confirm training succeeded
# Replace <job-name> with the actual job name from kubectl get jobs
kubectl logs -f job/<job-name>
```

---

## Task 2: Backend Inference Service with Load Balancing

### Build and push the backend image

```powershell
# Build the Docker image for the Flask backend inference server
docker build -t jzembower10/flask-backend:1.0.0 -f Dockerfile.backend .
```

```powershell
# Push the backend image to Docker Hub
docker push jzembower10/flask-backend:1.0.0
```

### Deploy the backend

```powershell
# Apply the backend manifest, which creates:
#   - A Deployment with 2 replicas of the Flask backend
#   - A NodePort Service exposing the backend on port 30080
kubectl apply -f backend-deployment.yaml
```

### Access the service

```powershell
# Create a tunnel from localhost to the Minikube NodePort service
# This gives you a local URL (e.g., http://127.0.0.1:57265) to reach the backend
# IMPORTANT: Keep this terminal open -- the tunnel dies if you close it
minikube service flask-backend-service
```

### Verify load balancing

```powershell
# Send a GET request to the /model-info endpoint
# Run this multiple times -- the "host" field should alternate between two pod names,
# proving Kubernetes distributes traffic across replicas
Invoke-WebRequest -UseBasicParsing -Uri http://127.0.0.1:57265/model-info | Select-Object -ExpandProperty Content
```

```powershell
# Send a POST request to the /predict endpoint with sample user features
# The response includes the predicted engagement_score and the serving pod's hostname
Invoke-WebRequest -UseBasicParsing -Uri http://127.0.0.1:57265/predict -Method POST -ContentType "application/json" -Body '{"avg_session_duration":30,"visits_per_week":14,"response_rate":4,"feature_usage_depth":6,"user_id":34}' | Select-Object -ExpandProperty Content
```

> **Note on PowerShell**: `curl` in PowerShell is an alias for `Invoke-WebRequest`, which does not
> support Unix-style curl flags like `--location` or `--request`. Use `Invoke-WebRequest` with
> `-UseBasicParsing` or use `curl.exe` to invoke the real curl binary.

---

## Task 3: preStop Lifecycle Hook

### Redeploy with the lifecycle hook

```powershell
# Re-apply the backend manifest after adding the preStop lifecycle hook configuration
kubectl apply -f backend-deployment.yaml
```

### Watch logs and trigger a rollout restart

```powershell
# In one terminal: stream logs from all backend pods in real-time
kubectl logs -l app=flask-backend -f
```

```powershell
# In a separate terminal: trigger a rolling restart to replace all pods
# This causes Kubernetes to terminate old pods (firing the preStop hook) and start new ones
kubectl rollout restart deployment/flask-backend-deployment
```

### Expected log output

After the rollout restart, the logs should show this sequence for each pod:

```
preStop signal received (SIGUSR1). Host preparing for shutdown: <pod-name>. Last model training time: <timestamp>
SIGTERM received. Host being terminated: <pod-name>. Last model training time: <timestamp>
```

This confirms the preStop hook (SIGUSR1) fires **before** Kubernetes sends SIGTERM.

---

## Useful Troubleshooting Commands

```powershell
# Open the Minikube web dashboard to visually monitor pods, deployments, and services
minikube dashboard
```

```powershell
# Check the Minikube IP address (useful for direct NodePort access)
minikube ip
```

```powershell
# View logs for all backend pods
kubectl logs -l app=flask-backend -f
```

```powershell
# List all pods in the default namespace to check status
kubectl get pods
```
