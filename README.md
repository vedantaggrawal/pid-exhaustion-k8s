# PID Exhaustion in Kubernetes — A Hands-On Lab

> I broke my Kubernetes pod with 9,395 zombies. Here's what actually happens during PID exhaustion — and why your node might be next.

A reproducible lab that demonstrates PID exhaustion in Kubernetes using a Kind cluster. Covers zombie processes, process leaks, PID namespace vs cgroup isolation, and why running containers without a proper init system is a ticking time bomb.

**GitHub**: [vedantaggrawal/pid-exhaustion-k8s](https://github.com/vedantaggrawal/pid-exhaustion-k8s)  

## What This Lab Demonstrates

| Pod | Mechanism | What Breaks |
|-----|-----------|-------------|
| `zombie-factory` | Python `os.fork()` without `os.wait()` — children become permanent zombies | Pod hits `pids.max`, `fork()` returns `EAGAIN` |
| `process-leak` | Busybox spawns `sleep infinity` — immortal processes pile up | Pod hits `pids.max`, crashes, enters `CrashLoopBackOff` |
| `innocent-bystander` | Normal pod that periodically tries to `fork()` | Stays healthy — proves cgroup isolation works |

## Prerequisites

- [Docker Desktop](https://docs.docker.com/desktop/)
- [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)

## Quick Start

```bash
# 1. Create a Kind cluster (skip if you already have one)
kind create cluster --name pid-lab

# 2. Deploy the lab
kubectl apply -f pid-exhaustion.yaml

# 3. Watch the zombie-factory hit PID exhaustion (~8-10 minutes)
kubectl logs -n pid-exhaustion zombie-factory -f

# 4. Watch the process-leak crash
kubectl logs -n pid-exhaustion process-leak -f

# 5. Confirm the bystander is unaffected
kubectl logs -n pid-exhaustion innocent-bystander -f
```

## Observing the Damage

### Pod-Level Commands

```bash
# Watch zombie accumulation in real-time
kubectl logs -n pid-exhaustion zombie-factory -f

# Watch process accumulation
kubectl logs -n pid-exhaustion process-leak -f

# Confirm bystander isolation
kubectl logs -n pid-exhaustion innocent-bystander -f

# Pod status overview
kubectl -n pid-exhaustion get pods -w

# Try to exec into the broken pod (will fail after PID exhaustion)
kubectl exec -n pid-exhaustion zombie-factory -- ps aux
```

### Node-Level Commands

These commands inspect the Kind node directly to see the real impact on the host:

```bash
# Get the Kind node name
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')

# ---- PID counts on the node ----

# Total processes on the node (includes ALL containers)
docker exec $NODE sh -c 'ls -d /proc/[0-9]* | wc -l'

# Count zombie processes on the node
docker exec $NODE sh -c 'cat /proc/*/stat 2>/dev/null | grep -c ") Z "'

# Node's PID max
docker exec $NODE cat /proc/sys/kernel/pid_max

# ---- Cgroup inspection ----

# Find the zombie-factory container's cgroup and check its PID limit
ZF_CID=$(kubectl -n pid-exhaustion get pod zombie-factory \
  -o jsonpath='{.status.containerStatuses[0].containerID}' | sed 's|containerd://||')

# Current PID usage vs limit for zombie-factory
docker exec $NODE sh -c "
  CGROUP_DIR=\$(dirname \$(find /sys/fs/cgroup -name 'pids.max' -path '*$ZF_CID*' 2>/dev/null | head -1))
  echo 'pids.current:' \$(cat \$CGROUP_DIR/pids.current)
  echo 'pids.max:    ' \$(cat \$CGROUP_DIR/pids.max)
  echo 'pids.peak:   ' \$(cat \$CGROUP_DIR/pids.peak 2>/dev/null || echo N/A)
"

# Same for process-leak
PL_CID=$(kubectl -n pid-exhaustion get pod process-leak \
  -o jsonpath='{.status.containerStatuses[0].containerID}' | sed 's|containerd://||')

docker exec $NODE sh -c "
  CGROUP_DIR=\$(dirname \$(find /sys/fs/cgroup -name 'pids.max' -path '*$PL_CID*' 2>/dev/null | head -1))
  echo 'pids.current:' \$(cat \$CGROUP_DIR/pids.current)
  echo 'pids.max:    ' \$(cat \$CGROUP_DIR/pids.max)
"

# ---- Full node PID summary ----
docker exec $NODE sh -c '
  echo "=== Node PID Summary ==="
  echo "pid_max:      $(cat /proc/sys/kernel/pid_max)"
  total=$(ls -d /proc/[0-9]* 2>/dev/null | wc -l)
  zombies=$(cat /proc/*/stat 2>/dev/null | grep -c ") Z ")
  echo "total procs:  $total"
  echo "zombies:      $zombies"
  echo "live procs:   $((total - zombies))"
  echo "available:    $(($(cat /proc/sys/kernel/pid_max) - total))"
'

# ---- Process list (see zombies marked as Z) ----
docker exec $NODE sh -c 'ps -eo pid,ppid,stat,comm | grep " Z" | head -20'
```

### Kubelet PID Limit Configuration

```bash
# Check what PID limit the kubelet is enforcing per pod
docker exec $NODE sh -c 'cat /var/lib/kubelet/config.yaml' | grep -i pid

# Check the kubelet's cgroup hierarchy
docker exec $NODE sh -c 'cat /sys/fs/cgroup/kubelet.slice/pids.max'
```

## Understanding the Results

### What You'll See

**zombie-factory** (Python, no init):
```
[zombie-factory] forked=9390  zombies=9389  total_procs=9393
[zombie-factory] forked=9395  zombies=9394  total_procs=9396
[zombie-factory] fork() FAILED: [Errno 11] Resource temporarily unavailable (PID exhaustion!)
```

**process-leak** (busybox, no init):
```
[process-leak] spawned=310  live_procs=315
[process-leak] spawned=320  live_procs=325
# then crash → CrashLoopBackOff
```

**innocent-bystander**:
```
[bystander] fork works fine - Sun Apr 19 15:44:46 UTC 2026
```

**Node-level**:
```
pid_max:      4194304
total procs:  9507
zombies:      9395
live procs:   112
available:    4184797    ← node is fine, cgroup contained the blast
```

## How It Works

### PID Namespaces vs Cgroups

```
                    ┌─────────────────────────────────┐
                    │         Node (kernel)            │
                    │    pid_max = 4,194,304           │
                    │                                  │
   ┌────────────────┼──────────┐  ┌───────────────────┐
   │  zombie-factory cgroup    │  │  bystander cgroup  │
   │  pids.max = 9396         │  │  pids.max = 9396   │
   │  pids.current = 9396 ❌  │  │  pids.current = 3 ✅│
   │                          │  │                     │
   │  ┌─ PID NS ──────────┐  │  │  ┌─ PID NS ─────┐  │
   │  │ PID 1: python      │  │  │  │ PID 1: sh    │  │
   │  │ PID 3: [zombie]    │  │  │  │              │  │
   │  │ PID 4: [zombie]    │  │  │  └──────────────┘  │
   │  │ ...9393 zombies    │  │  │                     │
   │  └────────────────────┘  │  │                     │
   └──────────────────────────┘  └─────────────────────┘
```

- **PID Namespace**: Cosmetic isolation — hides processes between containers
- **PID Cgroup** (`pids.max`): Real resource isolation — limits how many PIDs a pod can use
- **Without cgroup limits**: One bad pod exhausts the node's global PID table → kubelet breaks → node goes `NotReady`

### Why Busybox Reaps but Python Doesn't

| Runtime as PID 1 | Reaps Zombies? | Why |
|-------------------|:-:|---|
| busybox sh (ash) | Yes | Built-in job control calls `waitpid()` |
| bash | Yes | Job control reaps background jobs |
| python | **No** | `os.fork()` with no auto-`wait()` |
| node / java / go | **No** | No built-in child reaping |
| tini / dumb-init | Yes | Purpose-built init systems |

## The Fixes

### 1. Use a Proper Init System

```dockerfile
# Option A: tini
RUN apt-get update && apt-get install -y tini
ENTRYPOINT ["tini", "--"]
CMD ["python", "app.py"]

# Option B: dumb-init
RUN apt-get update && apt-get install -y dumb-init
ENTRYPOINT ["dumb-init", "--"]
CMD ["python", "app.py"]
```

### 2. Share the Process Namespace

```yaml
spec:
  shareProcessNamespace: true   # pause container becomes PID 1 and reaps
  containers:
  - name: app
    image: my-app:latest
```

### 3. Set Pod PID Limits

```yaml
# kubelet config
podPidsLimit: 1024
```

### 4. Monitor

- `container_processes` metric in cAdvisor
- `pids.current` vs `pids.max` in cgroup
- Alert on zombie count trending upward

## Cleanup

```bash
kubectl delete namespace pid-exhaustion

# If you created a cluster just for this lab:
kind delete cluster --name pid-lab
```

## Tested On

- Kind v0.27+, Kubernetes v1.35.0
- containerd 2.2.0, Linux 6.10.14
- macOS (Docker Desktop) and Linux hosts

## License

MIT
