# Kubernetes CrashLoopBackOff & OOMKilled — Real Production Fixes

> A field guide by engineers who've debugged these at 2am — not a docs rewrite.

---

## Table of Contents

- [Understanding CrashLoopBackOff](#understanding-crashloopbackoff)
- [Diagnosing CrashLoopBackOff — Step by Step](#diagnosing-crashloopbackoff--step-by-step)
- [Top CrashLoopBackOff Causes & Fixes](#top-crashloopbackoff-causes--fixes)
- [Understanding OOMKilled](#understanding-oomkilled)
- [Diagnosing OOMKilled — Step by Step](#diagnosing-oomkilled--step-by-step)
- [Top OOMKilled Causes & Fixes](#top-oomkilled-causes--fixes)
- [Prevention Checklist](#prevention-checklist)
- [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Understanding CrashLoopBackOff

`CrashLoopBackOff` is **not an error itself** — it's Kubernetes telling you: *"Your container keeps crashing and I'm backing off retrying it."*

The backoff timer doubles each time: 10s → 20s → 40s → 80s → 160s → 300s (capped).

```bash
# Check pod status
kubectl get pods -n <namespace>

# You'll see something like:
# NAME                        READY   STATUS             RESTARTS   AGE
# my-app-6d4b9f8c7-xk2pq     0/1     CrashLoopBackOff   8          12m
```

The restart count is your first clue — a count of 8+ means it's been crashing for a while.

---

## Diagnosing CrashLoopBackOff — Step by Step

### Step 1 — Get the last logs before crash

```bash
# Current logs (may be empty if container crashed immediately)
kubectl logs <pod-name> -n <namespace>

# CRITICAL: logs from the PREVIOUS crashed container
kubectl logs <pod-name> -n <namespace> --previous
```

> **Pro tip:** `--previous` is the most overlooked flag. The current container may have no logs yet — the previous one has the crash reason.

### Step 2 — Describe the pod

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Look at:
- `Last State` — exit code of the previous container
- `Events` section at the bottom — scheduling and pull errors show here
- `Liveness probe` failures

### Step 3 — Check exit codes

| Exit Code | Meaning |
|-----------|---------|
| `0` | Clean exit — your app exited intentionally (misconfigured CMD?) |
| `1` | General application error — check app logs |
| `137` | OOMKilled (signal 9) — memory limit hit |
| `139` | Segmentation fault — application bug |
| `143` | SIGTERM not handled — graceful shutdown issue |
| `255` | App-level error (common in misconfigured entrypoints) |

```bash
# Get exit code quickly
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'
```

### Step 4 — Exec into a running container (if it stays up briefly)

```bash
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh
# or /bin/bash if available
```

---

## Top CrashLoopBackOff Causes & Fixes

### 1. Missing or wrong environment variables

**Symptom:** Exit code 1, logs show `KeyError`, `undefined variable`, or connection refused on startup.

```bash
# Check what env vars are set
kubectl exec <pod-name> -- env | sort

# Check what the deployment expects
kubectl get deployment <name> -o yaml | grep -A 20 env:
```

**Fix:**
```yaml
# Bad — hardcoded and missing
env:
  - name: DB_HOST
    value: ""

# Good — using a secret
env:
  - name: DB_HOST
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: host
```

---

### 2. Liveness probe misconfigured

**Symptom:** Pod starts fine, runs for 30–60s, then restarts repeatedly. Logs show the app is healthy but K8s kills it anyway.

```bash
kubectl describe pod <pod-name> | grep -A 10 "Liveness"
# Look for: Liveness probe failed
```

**Fix — give your app time to start:**
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30   # ← increase this, default is 0
  periodSeconds: 10
  failureThreshold: 3
  timeoutSeconds: 5          # ← increase if your health endpoint is slow
```

> **Production rule:** `initialDelaySeconds` should be at least 1.5× your app's average cold-start time.

---

### 3. Image pull errors masking as CrashLoopBackOff

```bash
kubectl describe pod <pod-name> | grep -A 5 Events
# Look for: ErrImagePull or ImagePullBackOff
```

**Fix:**
```yaml
# Add imagePullSecrets if using private registry
spec:
  imagePullSecrets:
    - name: regcred
  containers:
    - name: my-app
      image: private-registry.io/my-app:v1.2.3  # Always pin tags, never :latest in prod
```

---

### 4. ConfigMap or Secret not found

**Symptom:** Pod never starts, exit code 1 immediately.

```bash
kubectl describe pod <pod-name> | grep -i "Error\|Warning"
# Look for: secret "xxx" not found
```

**Fix — verify before deploying:**
```bash
# Check secret exists
kubectl get secret <secret-name> -n <namespace>

# Check configmap exists
kubectl get configmap <cm-name> -n <namespace>

# Check all referenced secrets in a deployment
kubectl get deployment <name> -o json | jq '.spec.template.spec.containers[].env[]?.valueFrom.secretKeyRef.name' | sort -u
```

---

### 5. Application exiting with code 0 (clean exit)

**Symptom:** Exit code 0 but K8s keeps restarting. Your app finishes and exits — but K8s expects it to run forever.

**Fix — your CMD should be a long-running process:**
```dockerfile
# Bad — runs and exits
CMD ["python", "migrate.py"]

# Good — runs forever
CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0"]
```

For init/migration jobs use a Kubernetes `Job` resource, not a `Deployment`.

---

## Understanding OOMKilled

`OOMKilled` (Out Of Memory Killed) means the Linux kernel killed your container because it exceeded the memory limit set in the pod spec. Exit code is always `137`.

```bash
# Confirm OOMKilled
kubectl describe pod <pod-name> | grep -A 5 "Last State"
# Output:
# Last State: Terminated
#   Reason: OOMKilled
#   Exit Code: 137
```

---

## Diagnosing OOMKilled — Step by Step

### Step 1 — Check current memory limits vs actual usage

```bash
# Current usage
kubectl top pod <pod-name> -n <namespace>

# What limits are set
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[0].resources}'
```

### Step 2 — Check node-level memory pressure

```bash
kubectl describe node <node-name> | grep -A 10 "Conditions"
# Look for: MemoryPressure = True

kubectl top nodes
```

### Step 3 — Get historical memory metrics

```bash
# If you have metrics-server installed
kubectl top pod <pod-name> --containers

# Check events for OOM pattern
kubectl get events -n <namespace> --sort-by='.lastTimestamp' | grep OOM
```

---

## Top OOMKilled Causes & Fixes

### 1. Memory limit set too low

**Fix — right-size your limits:**
```yaml
resources:
  requests:
    memory: "256Mi"    # What K8s reserves on the node
  limits:
    memory: "512Mi"    # Hard ceiling — OOMKill happens here

# Rule of thumb:
# requests = p50 memory usage under normal load
# limits = p99 memory usage under peak load + 20% buffer
```

> **Never set limits without first observing real usage.** Run without limits in staging, measure with `kubectl top`, then set limits with headroom.

---

### 2. JVM heap not bounded (Java apps)

**Symptom:** Java app OOMKilled even with generous limits. JVM ignores container limits by default in older versions.

```yaml
env:
  - name: JAVA_OPTS
    value: "-XX:MaxRAMPercentage=75.0 -XX:InitialRAMPercentage=50.0"
    # This tells JVM to use max 75% of container memory limit
```

For Java 11+:
```yaml
env:
  - name: JVM_OPTS
    value: "-XX:+UseContainerSupport -XX:MaxRAMPercentage=75"
```

---

### 3. Memory leak in application

**Symptom:** Memory grows slowly over hours/days, then OOMKilled. Restarting temporarily fixes it.

```bash
# Watch memory grow in real time
watch -n 5 kubectl top pod <pod-name>
```

**Fix — add a readiness probe + resource limits as a safety net while fixing the leak:**
```yaml
resources:
  limits:
    memory: "1Gi"
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 30
  failureThreshold: 3
```

Also consider horizontal pod autoscaling to spread load while you fix the root cause.

---

### 4. Node memory pressure killing pods

**Symptom:** OOMKilled but your pod's memory usage looks fine. The *node* is out of memory.

```bash
kubectl describe node <node-name> | grep -B 5 -A 15 "Allocated resources"
```

**Fix — use PodDisruptionBudgets and node affinity to prevent overcommit:**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: my-app
```

---

## Prevention Checklist

```
✅ Always set both requests AND limits on every container
✅ Never use :latest image tag in production
✅ Set initialDelaySeconds on liveness probes ≥ app startup time
✅ Use --previous flag when checking crash logs
✅ Pin ConfigMap and Secret names in deployments
✅ Use readinessProbe separately from livenessProbe
✅ Monitor memory trends, don't just react to OOMKills
✅ Use Jobs for one-off tasks, not Deployments
✅ Set JVM heap bounds explicitly for Java containers
✅ Test resource limits under real load in staging first
```

---

## Quick Reference Cheat Sheet

```bash
# === CrashLoopBackOff Toolkit ===

# Previous container logs (most important)
kubectl logs <pod> -n <ns> --previous

# Exit code
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'

# Full pod description
kubectl describe pod <pod> -n <ns>

# Events in namespace
kubectl get events -n <ns> --sort-by='.lastTimestamp'

# === OOMKilled Toolkit ===

# Memory usage now
kubectl top pod <pod> -n <ns> --containers

# Node memory
kubectl top nodes

# Check limits
kubectl get pod <pod> -o jsonpath='{.spec.containers[0].resources}'

# Watch memory over time
watch -n 5 kubectl top pod <pod> -n <ns>
```

---

## When This Becomes a Full-Time Job

Debugging CrashLoopBackOff and OOMKilled once is a learning exercise.  
Doing it across 50+ microservices, multiple clusters, and at 3am is a different story.

> **Sygitech's [Cloud Monitoring & Management Services](https://www.sygitech.com/cloud-monitoring-and-management.html)** provide 24/7 proactive monitoring, real-time alerting, and expert incident response for Kubernetes environments — so your team focuses on building, not firefighting.
>
> 👉 [Talk to our cloud engineers](https://www.sygitech.com/cloud-monitoring-and-management.html)

---

*Maintained by [Sygitech](https://www.sygitech.com) — Managed Cloud Services for Engineering Teams*  
*Found an issue or have a fix to add? Open a PR or start a Discussion.*
