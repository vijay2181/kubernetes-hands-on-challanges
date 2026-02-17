**Pod Lifecycle and Restart Policies**:

```markdown
# Pod Lifecycle & Restart Policies

## Step 1: Create the Pod YAML file

Create a file named `lifecycle-demo.yaml`:

apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  restartPolicy: OnFailure
  containers:
  - name: lifecycle-container
    image: busybox:latest
    command: ["/bin/sh"]
    args: ["-c", "echo 'Container started'; sleep 5; echo 'Simulating failure'; exit 1"]
```

## Step 2: Apply the Pod

```bash
kubectl apply -f lifecycle-demo.yaml
```

## Step 3: Observe the pod lifecycle

Watch the pod status in real-time:

```bash
kubectl get pods -w
```

You'll see output similar to:
```
NAME              READY   STATUS              RESTARTS   AGE
lifecycle-demo    0/1     ContainerCreating   0          2s
lifecycle-demo    1/1     Running             0          5s
lifecycle-demo    0/1     Error               0          12s
lifecycle-demo    0/1     CrashLoopBackOff    1          15s
lifecycle-demo    0/1     Running             1          30s
lifecycle-demo    0/1     Error               1          37s
lifecycle-demo    0/1     CrashLoopBackOff    2          40s
```

## Step 4: What happens and why

### What happens:
1. **ContainerCreating (2s)**: Pod is scheduled, container image is pulled
2. **Running (5s)**: Container starts, executes commands
3. **Error (12s)**: Container exits with code 1 (simulated failure)
4. **CrashLoopBackOff (15s)**: Kubernetes detects the failure pattern
5. **Running (30s)**: Pod is restarted (2nd attempt)
6. **Error/BackOff cycle continues** with increasing delays

### Why this happens:

**Reason 1: RestartPolicy determines behavior**
```yaml
restartPolicy: OnFailure
```
- `Always` (default): Always restart regardless of exit code
- `OnFailure`: Restart only if container exits with non-zero code
- `Never`: Never restart automatically

**Reason 2: The CrashLoopBackOff mechanism**
Kubernetes implements exponential backoff:
- First restart: immediate (0s delay)
- Second restart: 10s delay
- Third restart: 20s delay
- Fourth restart: 40s delay
- Fifth restart: 80s delay
- Maximum delay: 300s (5 minutes)

**Reason 3: Container exit codes matter**
- Exit code 0: Success (no restart with OnFailure)
- Exit code 1-255: Failure (restart with OnFailure)
- Exit code 137: OOMKilled (always restarts based on policy)

## Step 5: Check pod details

Get detailed information about the pod's state:

```bash
kubectl describe pod lifecycle-demo
```

Look for key sections:
```
Status:           Failed
Containers:
  lifecycle-container:
    State:          Terminated
      Reason:       Error
      Exit Code:    1
    Last State:     Terminated
      Reason:       Error
      Exit Code:    1
    Ready:          False
    Restart Count:  3
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  2m                default-scheduler  Successfully assigned...
  Normal   Pulled     2m                kubelet            Successfully pulled image
  Normal   Created    2m                kubelet            Created container
  Normal   Started    2m                kubelet            Started container
  Normal   Pulled     105s              kubelet            Container image already present
  Normal   Created    105s              kubelet            Created container
  Normal   Started    105s              kubelet            Started container
  Warning  BackOff    75s (x5 over 95s) kubelet            Back-off restarting failed container
```

## Step 6: Experiment with different restart policies

### Try 1: Always restart (default)
Create `lifecycle-always.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-always
spec:
  restartPolicy: Always
  containers:
  - name: always-container
    image: busybox:latest
    command: ["/bin/sh"]
    args: ["-c", "echo 'Running once'; exit 0"]  # Even exit 0 triggers restart!
```

Apply and observe:
```bash
kubectl apply -f lifecycle-always.yaml
kubectl get pods -w
```
Notice: Even successful completion (exit 0) causes restarts!

### Try 2: Never restart
Create `lifecycle-never.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-never
spec:
  restartPolicy: Never
  containers:
  - name: never-container
    image: busybox:latest
    command: ["/bin/sh"]
    args: ["-c", "echo 'Running once'; sleep 3; exit 1"]
```

Apply and observe:
```bash
kubectl apply -f lifecycle-never.yaml
kubectl get pods -w
```
Notice: Pod stays in "Completed" or "Error" state with 0 restarts

## Step 7: Check pod status categories

After running all experiments, check different statuses:

```bash
# View all pods
kubectl get pods

# Filter by status
kubectl get pods --field-selector status.phase=Running
kubectl get pods --field-selector status.phase=Failed

# Get pod completion status
kubectl get pods -o wide
```

You'll see various phases:
- **Pending**: Pod accepted but not running
- **Running**: At least one container running
- **Succeeded**: All containers terminated successfully
- **Failed**: All containers terminated with failure
- **Unknown**: Pod state unknown

## Step 8: Container states deep dive

While a pod is running, examine container states:

```bash
# In one terminal, start a long-running pod
kubectl run nginx-demo --image=nginx --restart=Never

# In another terminal, watch container states
kubectl get pod nginx-demo -o jsonpath='{.status.containerStatuses[0].state}'
```

Container states include:
- **Waiting**: Not yet running (pulling image, waiting for conditions)
- **Running**: Executing normally
- **Terminated**: Completed or failed with exit code

## Summary:

The pod lifecycle demonstrates:
1. **Pod phases** (Pending → Running → Succeeded/Failed)
2. **Restart policies** (Always, OnFailure, Never) and their impact
3. **CrashLoopBackOff** as a protection mechanism
4. **Container states** and transitions
5. **Kubernetes self-healing** behavior

This concept is fundamental to understanding how Kubernetes maintains application availability and how to configure appropriate restart behaviors for different workload types (batch jobs vs. long-running services).
```

This task demonstrates:
- **Pod lifecycle phases** (Pending → Running → Failed)
- **Restart policies** (Always, OnFailure, Never)
- **CrashLoopBackOff** mechanism
- **Container states** (Waiting, Running, Terminated)
- **Exponential backoff** behavior
- **Exit codes** and their significance

It's simple to execute but shows the fundamental self-healing behavior of Kubernetes!
