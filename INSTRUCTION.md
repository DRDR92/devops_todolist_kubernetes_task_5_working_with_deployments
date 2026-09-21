# Kubernetes Deployment Instructions for Django ToDo App

## Prerequisites
- Kubernetes cluster v1.19+ (with metrics-server for HPA)
- `kubectl` configured
- Docker image built and pushed (e.g., `docker push <your-username>/django-todo:latest`)

## Deployment Steps

### 1. Create Namespace
```bash
kubectl apply -f namespace.yml
```

### 2. Deploy Application
```bash
kubectl apply -f deployment.yml
kubectl apply -f service.yml
kubectl apply -f hpa.yml
```

### 3. Verify Deployment
```bash
# Check pods are running
kubectl get pods -n mateapp
kubectl get deployment -n mateapp
kubectl get hpa -n mateapp

# View logs
kubectl logs -n mateapp deployment/todo-app -f

# Check service status
kubectl get svc -n mateapp
```

## Accessing the Application

### Local Development (minikube)
```bash
minikube service todo-app-service -n mateapp
```
This will automatically open the app in your browser at the service's external IP.

### Kubernetes Service (any cluster)
```bash
kubectl port-forward -n mateapp svc/todo-app-service 8000:80
```
Then access at `http://localhost:8000`

### External Access (if LoadBalancer is supported)
```bash
kubectl get svc -n mateapp
```
Look for the EXTERNAL-IP and access at `http://<EXTERNAL-IP>`

### NodePort Access (if LoadBalancer not available)
Change service type from `LoadBalancer` to `NodePort` in `service.yml`, then:
```bash
kubectl get svc -n mateapp
# Access at http://<NODE-IP>:<NODE-PORT>
```

---

## Configuration Explanations

### Resource Requests and Limits

#### Chosen Values:
- **Requests**: 50m CPU, 128Mi Memory
- **Limits**: 250m CPU, 512Mi Memory

#### Rationale:

**Why these numbers:**
1. **Django with runserver is lightweight** — it's a development server not optimized for production, but uses minimal resources
2. **Idle state baseline** — based on typical Django app idle consumption (~30-50m CPU, 100-150Mi memory)
3. **Headroom for requests** — limits allow 5x burst capacity (250m) for handling traffic spikes

**Requests vs Limits:**
- **Requests** (50m, 128Mi): Tell Kubernetes "reserve at least this much for each pod"
  - Used for scheduler decisions (placement, resource allocation)
  - In idle state with 2 pods: 100m CPU, 256Mi memory total reserved
  - Allows efficient bin-packing of other workloads
  
- **Limits** (250m, 512Mi): Hard ceiling to prevent runaway processes
  - Pod killed if it exceeds memory limit
  - CPU throttled if exceeds limit (doesn't kill, just slows down)
  - Protects cluster from one misbehaving pod taking all resources

**How limits support the task:**
- 2 pods at idle: ~100-150Mi memory used
- Room for 3-4 additional pods before hitting memory constraints on small cluster
- Prevents accidental DoS from Django memory leak

---

### Horizontal Pod Autoscaler (HPA) Configuration

#### Chosen Values:
```yaml
minReplicas: 2      # Minimum pods running
maxReplicas: 5      # Maximum pods allowed
CPU target: 70%     # Scale up when CPU > 70%
Memory target: 80%  # Scale up when memory > 80%
```

#### Rationale:

**Minimum = 2:**
- Task requirement: "2 pods running in idle state"
- Provides basic redundancy (one pod fails, app still online)
- Meets HA requirement for production-grade apps
- Pair with rolling updates = zero-downtime deployments

**Maximum = 5:**
- Prevents runaway scaling (cost control)
- Reasonable for this lightweight app (5 × 250m = 1.25 CPU, 5 × 512Mi = 2.56Gi max)
- Matches typical small cluster resources
- Can scale 2.5x for handling traffic spikes

**CPU Target = 70%:**
- Conservative threshold (not waiting until 100%)
- Gives ~30% headroom before hitting pod limits
- Scaling takes 15-30s; 70% ensures new pods ready before saturation
- For runserver: adequate to handle typical Django request processing

**Memory Target = 80%:**
- Higher than CPU (memory scaling is slower, can't be throttled like CPU)
- Leaves 20% buffer to avoid OOMKill (Out Of Memory Kill)
- Django memory rarely spikes suddenly; 80% gives time to react
- Prevents cascade failure from memory exhaustion

**Scaling Behavior:**
```yaml
scaleUp:    # Fast scaling when demand increases
  - Can add 100% of current pods every 30 seconds
  - Example: 2 pods → 4 pods → 8 pods (if load keeps rising)
  - No stabilization wait (immediate response)

scaleDown:  # Slow scaling when demand decreases
  - Removes 50% of excess pods every 60 seconds
  - Example: 5 pods → 3 pods → 2 pods (gradual)
  - 300s stabilization prevents flapping (quick up/down cycling)
  - Protects against temporary traffic dips
```

**Why these metrics (CPU + Memory):**
- CPU alone: scales too aggressively if app is memory-bound
- Memory alone: scales too slowly if app is CPU-bound
- Both: catches different bottlenecks (e.g., calculation vs data loading)
- Either metric crossing threshold triggers scale-up

---

### Rolling Update Strategy

#### Chosen Configuration:
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1          # Allow 1 extra pod during update
    maxUnavailable: 0    # Never take pods offline
```

#### Rationale:

**RollingUpdate vs other strategies:**

| Strategy | Behavior | Downtime | Resource Cost | Use Case |
|----------|----------|----------|---------------|----------|
| RollingUpdate | Replace pods one-by-one | None (zero-downtime) | +1 pod temporarily | Production, user-facing |
| Recreate | Kill all, start new | Yes (~1min) | Same as running | Dev/test, maintenance windows |
| Blue-Green | Run old and new in parallel | None | 2x resources | Large deployments, rollback safety |
| Canary | Gradually increase traffic % | None | +1 pod | Critical apps, gradual rollouts |

**For this project: RollingUpdate** (recommended)
- Ensures zero-downtime during updates
- Minimal extra resource cost (only 1 surge pod)
- Simple and safe for Django apps
- Sufficient for educational/demonstration purposes

**maxSurge = 1:**
- Allows 1 additional pod temporarily (e.g., 2 pods → 3 pods briefly)
- Higher values: faster updates but more resource cost
- Lower (0): would require pods to be terminated before new ones start (causes downtime)
- Value of 1: good balance — update takes ~30-60s per pod

**maxUnavailable = 0:**
- Guarantees minimum 2 pods always running (never drop below)
- **Never** takes old pods offline until new ones are ready and passing health checks
- Zero-downtime requirement (traffic never blocked)
- Requests must wait for available pod, but don't fail

**Example rollout sequence (2 pods → 3 pods → 2 pods):**
```
Step 1: Original state (2 pods)
  Pod-A (old) — handling traffic
  Pod-B (old) — handling traffic

Step 2: Create new pod
  Pod-A (old) — handling traffic
  Pod-B (old) — handling traffic
  Pod-C (new) — starting up, running health checks

Step 3: Pod-C ready, terminate Pod-A
  Pod-C (new) — handling traffic
  Pod-B (old) — handling traffic

Step 4: Create new pod
  Pod-C (new) — handling traffic
  Pod-B (old) — handling traffic
  Pod-D (new) — starting up, health checks

Step 5: Pod-D ready, terminate Pod-B
  Pod-C (new) — handling traffic
  Pod-D (new) — handling traffic
  
Result: Full update, zero traffic loss, 2 pods always running
```

**Why these numbers for this project:**
- App is stateless (Django with SQLite or no persistent state)
- No complex initialization required
- Health checks work reliably (HTTP GET on port 8080)
- Users won't notice 30-60s update process

---

## Monitoring & Troubleshooting

### Check HPA Status
```bash
kubectl describe hpa todo-app-hpa -n mateapp
```
Shows current CPU/memory usage vs targets, scaling events history.

### View Scaling Events
```bash
kubectl get events -n mateapp --sort-by='.lastTimestamp' | grep todo-app
```

### Debug Pod Issues
```bash
# Pod logs
kubectl logs -n mateapp <pod-name>

# Pod describe (resource limits, events)
kubectl describe pod -n mateapp <pod-name>

# Exec into pod
kubectl exec -it -n mateapp <pod-name> -- /bin/bash
```

### Scaling Manually (for testing)
```bash
# Force update image (triggers rollout)
kubectl set image deployment/todo-app -n mateapp \
  todo-app=<your-username>/django-todo:v2

# Scale manually (HPA will override if metrics exceed threshold)
kubectl scale deployment todo-app -n mateapp --replicas=4
```

---

## Cleanup

```bash
# Remove HPA, deployment, service
kubectl delete deployment,svc,hpa -n mateapp -l app=todo-app

# Remove entire namespace
kubectl delete namespace mateapp
```

---

## Next Steps (for production)

1. **Replace runserver** with gunicorn/uWSGI for performance
2. **Externalize database** (PostgreSQL/MySQL instead of SQLite)
3. **Add Ingress** for HTTPS and path-based routing
4. **Configure PersistentVolumes** for database and media files
5. **Add resource quotas** at namespace level
6. **Implement dedicated logging/monitoring** (Prometheus, Grafana)
7. **Use resource metrics more granularly** (per-container limits)
