# KEDA GPU Autoscaler

Event-driven autoscaling of GPU workloads on Kubernetes using KEDA and Redis.
Pods scale 0→N based on queue depth and return to zero when idle.

## Architecture
Redis queue → KEDA ScaledObject → GPU worker Deployment (nodeSelector: gpu=true)

## What it does
- Scale-to-zero when queue is empty (zero idle GPU cost)
- 1 pod per 2 queued jobs (configurable via listLength)
- Constrained to GPU-labelled nodes via nodeSelector

  # ── Step 1: Install Helm ──────────────────────────────────────────
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# ── Step 2: Install KEDA ─────────────────────────────────────────
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda \
  --namespace keda \
  --create-namespace \
  --wait

# Verify (you should see 3 pods: operator, metrics-apiserver, admission-webhooks)
kubectl get pods -n keda

# ── Step 3: Deploy Redis (our job queue) ─────────────────────────
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  selector:
    matchLabels: {app: redis}
  template:
    metadata:
      labels: {app: redis}
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  selector: {app: redis}
  ports:
  - port: 6379
    targetPort: 6379
EOF

kubectl rollout status deployment/redis



Phase 2 — Simulate GPU node 

# Label the worker node to simulate a GPU node
# (in real prod this would be nvidia.com/gpu=true from device plugin)
kubectl label node node01 gpu=true

# Verify
kubectl get nodes --show-labels | grep gpu


Phase 3 — Deploy the "GPU worker" Deployment

kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-worker
  labels:
    app: gpu-worker
spec:
  replicas: 0          # KEDA will control this
  selector:
    matchLabels: {app: gpu-worker}
  template:
    metadata:
      labels: {app: gpu-worker}
    spec:
      nodeSelector:
        gpu: "true"    # Only schedule on our "GPU" node (simulated)
      containers:
      - name: worker
        image: busybox
        command: ["sh", "-c", "echo GPU worker running job; sleep 30"]
        resources:
          requests:
            cpu: "100m"
            memory: "64Mi"
EOF


Phase 4 — Create the KEDA ScaledObject 

kubectl apply -f - <<EOF
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: gpu-worker-scaler
spec:
  scaleTargetRef:
    name: gpu-worker
  minReplicaCount: 0       # scale to ZERO when queue empty
  maxReplicaCount: 5
  pollingInterval: 5       # check queue every 5 seconds
  cooldownPeriod: 10       # scale down 10s after queue empties
  triggers:
  - type: redis
    metadata:
      address: redis.default.svc.cluster.local:6379
      listName: gpu-jobs          # Redis list KEDA watches
      listLength: "2"             # 1 pod per 2 jobs
EOF

# Verify ScaledObject is active
kubectl get scaledobject gpu-worker-scaler
# STATUS should show: Active=True

Phase 5 — Push jobs & watch autoscaling live 
Terminal 1 — Watch pods in real time:
watch -n 2 "kubectl get pods -l app=gpu-worker && echo '---' && kubectl get hpa"

Terminal 2 — Push jobs to the queue:
# Get into Redis
REDIS_POD=$(kubectl get pod -l app=redis -o jsonpath='{.items[0].metadata.name}')

# Push 6 jobs → should trigger 3 pods (6 jobs ÷ 2 = 3)
kubectl exec -it $REDIS_POD -- redis-cli RPUSH gpu-jobs \
  job-001 job-002 job-003 job-004 job-005 job-006

# Check queue depth
kubectl exec -it $REDIS_POD -- redis-cli LLEN gpu-jobs

# Watch KEDA scale up in Terminal 1 — takes ~10 seconds

# Now drain the queue (simulate jobs finishing)
kubectl exec -it $REDIS_POD -- redis-cli DEL gpu-jobs

# Watch pods scale back to 0 after cooldownPeriod (10s)

Phase 6 — Verify

# 1. Check KEDA operator logs (see scaling decisions)
kubectl logs -n keda -l app=keda-operator --tail=20

# 2. Describe the HPA KEDA created automatically
kubectl describe hpa keda-hpa-gpu-worker-scaler

# 3. Check ScaledObject events
kubectl describe scaledobject gpu-worker-scaler

# 4. Confirm nodeSelector worked (all pods on GPU node)
kubectl get pods -l app=gpu-worker -o wide
# NODE column should show: node01

