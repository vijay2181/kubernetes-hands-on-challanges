# Resource Constraints & OOMKill

## Step 1: Create the Deployment YAML file

Create a file named `memory-hungry-calculator.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: memory-hungry-calculator
spec:
  replicas: 1
  selector:
    matchLabels:
      app: memory-hungry-calculator
  template:
    metadata:
      labels:
        app: memory-hungry-calculator
    spec:
      containers:
      - name: calculator
        image: alpine:latest
        command: ["sh", "-c", "mkfifo /tmp/f && cat /tmp/f"]
        resources:
          requests:
            memory: "10Mi"
          limits:
            memory: "50Mi"
```

## Step 2: Apply the Deployment

```bash
kubectl apply -f memory-hungry-calculator.yaml
```

## Step 3: Observe what happens to the pod

Check the status of the pod:

```bash
kubectl get pods -w
```

You'll see output similar to:
```
NAME                                         READY   STATUS      RESTARTS   AGE
memory-hungry-calculator-xxxx-xxxx          0/1     OOMKilled   0          5s
```

## Step 4: What happens and why

### What happens:
1. The pod starts but quickly gets terminated with status `OOMKilled` (Out Of Memory Killed)
2. Kubernetes will keep restarting the pod (CrashLoopBackOff), but it will continue to get OOMKilled each time

### Why this happens:

**Reason 1: The container command itself creates a memory issue**
```bash
mkfifo /tmp/f && cat /tmp/f
```
This command:
- Creates a named pipe at `/tmp/f`
- Runs `cat /tmp/f` which tries to read from the pipe
- Since nothing is writing to the pipe, `cat` will block indefinitely waiting for input
- However, even a simple `cat` process requires some minimal memory to run

**Reason 2: The memory limit is extremely restrictive**
- Alpine Linux container has a minimal memory footprint, but even the base OS + shell + cat process needs some memory
- The memory limit of 50Mi is very low, but the actual issue is...
- **The memory request of 10Mi is critically low** - this is the guaranteed memory the scheduler reserves for the pod

**Reason 3: Memory overhead not accounted for**
Even a minimal Alpine container running just `sh` and `cat` needs more than 10Mi. The memory usage includes:
- Alpine base OS: ~5-8MB
- Shell process (`sh`): Additional memory
- `cat` process: Additional memory
- Container runtime overhead
- Kernel data structures

**Reason 4: OOMKiller trigger**
When the container's actual memory usage exceeds the 50Mi limit (or when the node is under memory pressure and the container exceeds its request), the Linux OOM killer terminates the process.

## Additional verification:

Check pod details for more information:
```bash
kubectl describe pod memory-hungry-calculator-xxxx-xxxx
```

Look for events like:
```
Events:
  Type     Reason     Age               From               Message
  ----     ------     ----              ----               -------
  Normal   Scheduled  <some-time>       default-scheduler  Successfully assigned...
  Normal   Pulled     <some-time>       kubelet            Container image "alpine:latest" already present on machine
  Normal   Created    <some-time>       kubelet            Created container calculator
  Normal   Started    <some-time>       kubelet            Started container calculator
  Warning  OOMKilled  <some-time>       kubelet            Container calculator was OOMKilled
```

## Summary:
The pod fails because the memory request (10Mi) is too low for even the most basic Alpine container to start and run processes. The container gets OOMKilled almost immediately after starting because it can't operate within the requested memory constraints. Kubernetes will keep trying to restart it, leading to a CrashLoopBackOff state.
