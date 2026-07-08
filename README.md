# KEDA GPU Autoscaler

Event-driven autoscaling of GPU workloads on Kubernetes using KEDA and Redis.

Pods scale **0→N** based on queue depth and return to zero when idle.

## Architecture

```
Redis queue → KEDA ScaledObject → GPU worker Deployment (nodeSelector: gpu=true)
```

## What it does

- Scale-to-zero when queue is empty (zero idle GPU cost)
- 1 pod per 2 queued jobs (configurable via `listLength`)
- Constrained to GPU-labelled nodes via `nodeSelector`

---

# Step 1: Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

---

# Step 2: Install KEDA

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

helm install keda kedacore/keda \
  --namespace keda \
  --create-namespace \
  --wait
```

Verify (you should see 3 pods: operator, metrics-apiserver, admission-webhooks)

```bash
kubectl get pods -n keda
```

---

# Step 3: Deploy Redis (our job queue)

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
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
  selector:
    app: redis
  ports:
    - port: 6379
      targetPort: 6379
EOF

kubectl rollout status deployment/redis
```

---

# Phase 2 — Simulate GPU node

Label the worker node to simulate a GPU node.

(In real production this would come from the NVIDIA device plugin.)

```bash
kubectl label node node01 gpu=true

kubectl get nodes --show-labels | grep gpu
```

---

# Phase 3 — Deploy the GPU worker Deployment

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-worker
  labels:
    app: gpu-worker
spec:
  replicas: 0
  selector:
    matchLabels:
      app: gpu-worker
  template:
    metadata:
      labels:
        app: gpu-worker
    spec:
      nodeSelector:
        gpu: "true"
      containers:
        - name: worker
          image: busybox
          command:
            - sh
            - -c
            - echo GPU worker running job; sleep 30
          resources:
            requests:
              cpu: "100m"
              memory: "64Mi"
EOF
```

---

# Phase 4 — Create the KEDA ScaledObject

```bash
kubectl apply -f - <<EOF
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: gpu-worker-scaler
spec:
  scaleTargetRef:
    name: gpu-worker
  minReplicaCount: 0
  maxReplicaCount: 5
  pollingInterval: 5
  cooldownPeriod: 10
  triggers:
    - type: redis
      metadata:
        address: redis.default.svc.cluster.local:6379
        listName: gpu-jobs
        listLength: "2"
EOF
```

Verify the ScaledObject is active.

```bash
kubectl get scaledobject gpu-worker-scaler
```

Expected status:

```
Active=True
```

---

# Phase 5 — Push jobs & watch autoscaling live

## Terminal 1

Watch pods and the generated HPA.

```bash
watch -n 2 "kubectl get pods -l app=gpu-worker && echo '---' && kubectl get hpa"
```

## Terminal 2

Get the Redis pod.

```bash
REDIS_POD=$(kubectl get pod -l app=redis -o jsonpath='{.items[0].metadata.name}')
```

Push 6 jobs (6 jobs ÷ 2 = 3 pods).

```bash
kubectl exec -it $REDIS_POD -- redis-cli RPUSH gpu-jobs \
  job-001 job-002 job-003 job-004 job-005 job-006
```

Check queue depth.

```bash
kubectl exec -it $REDIS_POD -- redis-cli LLEN gpu-jobs
```

Watch KEDA scale up (about 10 seconds).

Drain the queue.

```bash
kubectl exec -it $REDIS_POD -- redis-cli DEL gpu-jobs
```

Watch the deployment scale back to zero after the `cooldownPeriod` (10 seconds).

---

# Phase 6 — Verify

Check KEDA operator logs.

```bash
kubectl logs -n keda -l app=keda-operator --tail=20
```

Describe the HPA created by KEDA.

```bash
kubectl describe hpa keda-hpa-gpu-worker-scaler
```

Check ScaledObject events.

```bash
kubectl describe scaledobject gpu-worker-scaler
```

Confirm the `nodeSelector` worked.

```bash
kubectl get pods -l app=gpu-worker -o wide
```

Expected output:

```
NODE
node01
```
