# Solution for Liveness Probe Failure Scenario

## Step 1: Create the Deployment YAML

Create a file named `unhealthy-service.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: unhealthy-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: unhealthy-service
  template:
    metadata:
      labels:
        app: unhealthy-service
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        livenessProbe:
          exec:
            command:
            - sh
            - -c
            - curl -f http://localhost/health
          initialDelaySeconds: 10
          periodSeconds: 5
```

## Step 2: Apply the Deployment

```bash
kubectl apply -f unhealthy-service.yaml
```

## Step 3: Observe the pod lifecycle

Watch the pod status:

```bash
kubectl get pods -w
```

You'll see output like this:
```
NAME                                  READY   STATUS    RESTARTS   AGE
unhealthy-service-xxxxx-xxxxx         1/1     Running   0          12s
unhealthy-service-xxxxx-xxxxx         0/1     Running   1          45s
unhealthy-service-xxxxx-xxxxx         1/1     Running   1          47s
unhealthy-service-xxxxx-xxxxx         0/1     Running   2          80s
unhealthy-service-xxxxx-xxxxx         1/1     Running   2          82s
```

## Step 4: Get detailed pod information

```bash
kubectl describe pod unhealthy-service-xxxxx-xxxxx
```

## What happens and why:

### What happens:
1. **Pod starts normally**: The pod initially starts and shows `Running` status
2. **Initial delay**: The liveness probe waits 10 seconds before starting (as configured with `initialDelaySeconds: 10`)
3. **First liveness check fails**: After 10 seconds, the probe runs `curl -f http://localhost/health`
4. **Container restarts**: When the liveness probe fails, Kubernetes kills and restarts the container
5. **Cycle repeats**: The new container starts, runs for 10 seconds, fails the health check, and gets restarted again

### Why this happens:

**Reason 1: Nginx doesn't have a `/health` endpoint by default**
- The standard Nginx image doesn't have a `/health` endpoint configured
- The `curl -f` command fails because it gets a `404 Not Found` response
- The `-f` flag makes `curl` fail on HTTP error responses (like 404)

**Reason 2: Liveness probe configuration**
- `initialDelaySeconds: 10`: Waits 10 seconds before starting the first probe
- `periodSeconds: 5`: Runs the probe every 5 seconds
- The probe uses `exec` to run `curl` inside the container

**Reason 3: Kubernetes liveness probe behavior**
When a liveness probe fails:
1. Kubernetes kills the container
2. The container restarts (keeping the same pod)
3. The restart count increments
4. After multiple failures, the pod may enter `CrashLoopBackOff` state

## Step 5: Check events and logs

To understand better, check the events:

```bash
kubectl describe pod unhealthy-service-xxxxx-xxxxx | grep -A 10 Events
```

You'll see events like:
```
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  2m                 default-scheduler  Successfully assigned...
  Normal   Pulled     2m                 kubelet            Container image "nginx:alpine" already present
  Normal   Created    2m                 kubelet            Created container nginx
  Normal   Started    2m                 kubelet            Started container nginx
  Warning  Unhealthy  90s (x3 over 2m)   kubelet            Liveness probe failed: 
  Normal   Killing    90s                kubelet            Container nginx failed liveness probe, will be restarted
  Normal   Pulled     89s (x2 over 2m)   kubelet            Container image "nginx:alpine" already present
  Normal   Created    89s (x2 over 2m)   kubelet            Created container nginx
  Normal   Started    89s (x2 over 2m)   kubelet            Started container nginx
```

To see the actual error from curl:

```bash
# Get logs from the previous container (before restart)
kubectl logs unhealthy-service-xxxxx-xxxxx --previous
```

The logs won't show the curl error because it's from the liveness probe execution, but you can test manually:

```bash
# Exec into the container and test the command
kubectl exec -it unhealthy-service-xxxxx-xxxxx -- sh -c 'curl -v http://localhost/health'
```

This will show:
```
< HTTP/1.1 404 Not Found
```

## Summary:
The pod enters a restart loop because:
1. Nginx by default doesn't have a `/health` endpoint
2. The liveness probe checks for this non-existent endpoint every 5 seconds
3. When the probe fails (gets 404), Kubernetes restarts the container
4. This creates an infinite loop of: Start → Wait 10s → Health check fails → Restart

## To fix this:
You would need to either:
1. Configure Nginx with a proper health endpoint
2. Change the liveness probe to check an existing endpoint (like `/`)
3. Create a custom health check endpoint

Example fix (checking the root endpoint instead):
```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 10
  periodSeconds: 5
```
