# ☸️ Challenge 4: ConfigMap & Environment Variables

## 📋 Concept Overview

**ConfigMaps** are Kubernetes objects used to decouple configuration artifacts from container images. This challenge demonstrates how to inject configuration data into pods as environment variables, enabling you to build twelve-factor applications that store configuration separately from code.

**Key concepts covered:**
- Creating and managing ConfigMaps
- Injecting configuration as environment variables
- Updating configuration without rebuilding images
- Environment vs volume-based configuration
- Twelve-factor app principles in Kubernetes

---

## 🎯 Challenge Goal

Create a ConfigMap with application configuration settings and inject them as environment variables into a pod. Verify that the configuration is correctly passed and accessible within the container.

---

## 📝 Step-by-Step Guide

### Step 1: Create a ConfigMap

Create a file named `app-config.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: production
  LOG_LEVEL: debug
  API_TIMEOUT: "30"
  FEATURE_FLAG_NEW_UI: "true"
```

Apply the ConfigMap:

```bash
kubectl apply -f app-config.yaml
```

### Step 2: Create a Pod that uses the ConfigMap

Create a file named `config-demo-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-demo
spec:
  containers:
  - name: demo-container
    image: busybox:latest
    command: ["/bin/sh"]
    args: ["-c", "echo '=== Environment Variables ==='; env | grep -E 'APP_ENV|LOG_LEVEL|API_TIMEOUT|FEATURE_FLAG'; echo '=== Sleeping for 3600s ==='; sleep 3600"]
    env:
    - name: APP_ENV
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_ENV
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LOG_LEVEL
    - name: API_TIMEOUT
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: API_TIMEOUT
    - name: FEATURE_FLAG_NEW_UI
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: FEATURE_FLAG_NEW_UI
```

Apply the pod:

```bash
kubectl apply -f config-demo-pod.yaml
```

### Step 3: Observe the results

Check the pod logs:

```bash
kubectl logs config-demo
```

Expected output:
```
=== Environment Variables ===
APP_ENV=production
LOG_LEVEL=debug
API_TIMEOUT=30
FEATURE_FLAG_NEW_UI=true
=== Sleeping for 3600s ===
```

### Step 4: Verify environment variables inside the container

Exec into the running container:

```bash
kubectl exec -it config-demo -- /bin/sh
```

Inside the container, run:

```bash
env | grep -E 'APP_ENV|LOG_LEVEL|API_TIMEOUT|FEATURE_FLAG'
exit
```

---

## 🔍 What's Happening?

### The Mechanism:

1. **ConfigMap Creation**: You created a standalone configuration object containing key-value pairs
2. **Reference Binding**: The pod specification references specific keys from the ConfigMap
3. **Value Injection**: Kubernetes injects these values as environment variables when the container starts
4. **Runtime Access**: The container processes can access configuration via standard environment variable APIs

### Key Insights:

- **Decoupling**: Configuration is stored separately from the pod definition
- **Reusability**: Same ConfigMap can be used across multiple pods
- **Environment Parity**: Same image can behave differently in dev/staging/prod
- **No Rebuild Required**: Configuration changes don't require rebuilding images

---

## 🧪 Experiments & Variations

### Variation 1: Inject ALL ConfigMap keys at once

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-demo-envfrom
spec:
  containers:
  - name: demo-container
    image: busybox:latest
    command: ["sleep", "3600"]
    envFrom:
    - configMapRef:
        name: app-config
```

Apply and check:

```bash
kubectl apply -f config-demo-envfrom.yaml
kubectl exec config-demo-envfrom -- env
```

### Variation 2: Mount ConfigMap as a volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-demo-volume
spec:
  volumes:
  - name: config-volume
    configMap:
      name: app-config
  containers:
  - name: demo-container
    image: busybox:latest
    command: ["sleep", "3600"]
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
```

Check the mounted files:

```bash
kubectl exec config-demo-volume -- ls -la /etc/config/
kubectl exec config-demo-volume -- cat /etc/config/APP_ENV
```

### Variation 3: Update ConfigMap and observe behavior

```bash
# Update the ConfigMap
kubectl edit configmap app-config
# Change LOG_LEVEL: debug → LOG_LEVEL: warn

# Check existing pod (environment variables DON'T auto-update)
kubectl exec config-demo -- env | grep LOG_LEVEL
# Still shows 'debug'

# Check volume-mounted pod (files DO auto-update - eventually)
kubectl exec config-demo-volume -- cat /etc/config/LOG_LEVEL
# May show 'warn' after a delay (kubelet sync period)
```

### Variation 4: Imperative ConfigMap creation

```bash
# From literal values
kubectl create configmap literal-config \
  --from-literal=DB_HOST=localhost \
  --from-literal=DB_PORT=5432

# From a file
echo "max_connections=100" > db.properties
echo "shared_buffers=256MB" >> db.properties
kubectl create configmap file-config \
  --from-file=db.properties

# From a directory
mkdir config-files
echo "enabled=true" > config-files/feature-flags
echo "cache_ttl=3600" > config-files/cache-settings
kubectl create configmap dir-config \
  --from-file=config-files/
```

### Variation 5: Mix with hard-coded values

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-demo-mixed
spec:
  containers:
  - name: demo-container
    image: busybox:latest
    command: ["sleep", "3600"]
    env:
    - name: STATIC_VALUE
      value: "hard-coded"
    - name: APP_ENV
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_ENV
    envFrom:
    - configMapRef:
        name: app-config
```

---

## 🚨 Common Pitfalls & Solutions

| Pitfall | Symptom | Solution |
|---------|---------|----------|
| **ConfigMap doesn't exist** | Pod stuck in `ContainerCreating` | Create ConfigMap first or use `optional: true` |
| **Key doesn't exist** | Pod fails to start with error | Verify key names or use `optional: true` |
| **Environment variables don't update** | Old values still used | Use volume mounts for dynamic updates or restart pods |
| **Large ConfigMaps** | Pod startup delay | Keep ConfigMaps under 1MB limit |
| **Special characters in values** | Variables not set correctly | Use quotes or base64 encoding |
| **Case sensitivity** | Variables not found | Environment variables are case-sensitive |
| **Secret data exposure** | Sensitive data in logs | Use Secrets for sensitive data, not ConfigMaps |

---

## 📊 ConfigMap vs Other Approaches

| Approach | Use Case | Pros | Cons |
|----------|----------|------|------|
| **Environment Variables** | Simple config, 12-factor apps | Easy to use, language-agnostic | No auto-update, size limits |
| **Volume Mounts** | Dynamic config, large configs | Auto-update, supports large files | Filesystem overhead, delay in updates |
| **Hard-coded values** | Quick testing | Simple, no dependencies | Not portable, violates 12-factor |
| **Secrets** | Sensitive data | Encrypted, RBAC controls | More complex, size limits |
| **External config services** | Complex microservices | Centralized, dynamic | External dependency, latency |

---

## 💡 Real-World Use Cases

1. **Environment-specific configuration**
   ```yaml
   # Dev environment
   API_URL: https://dev-api.example.com
   
   # Prod environment (different ConfigMap)
   API_URL: https://api.example.com
   ```

2. **Feature flags**
   ```yaml
   NEW_PAYMENT_GATEWAY: "false"
   ENABLE_BETA_FEATURES: "true"
   MAINTENANCE_MODE: "false"
   ```

3. **Application tuning**
   ```yaml
   MAX_CONNECTIONS: "100"
   CACHE_TTL_SECONDS: "3600"
   LOG_LEVEL: "info"
   ```

4. **Third-party service endpoints**
   ```yaml
   REDIS_HOST: redis-service.prod.svc.cluster.local
   REDIS_PORT: "6379"
   DATABASE_URL: postgresql://user:password@postgres:5432/app
   ```

---

## 🎓 Key Takeaways

1. **Separation of concerns** - Keep configuration separate from container images
2. **Immutable infrastructure** - Same image, different configs for different environments
3. **Twelve-factor apps** - Store config in the environment
4. **Update strategies** - Volume mounts for dynamic updates, env vars for static config
5. **Security awareness** - Use Secrets for sensitive data, not ConfigMaps

---

## 🔗 Related Concepts

- **Secrets** - Similar to ConfigMaps but for sensitive data
- **Environment Variables** - Standard way to pass config in containers
- **Downward API** - Expose pod metadata as environment variables
- **Init Containers** - Generate configuration at startup
- **Helm Charts** - Template-based configuration management

---

## 📖 Commands Reference

```bash
# Create ConfigMap
kubectl create configmap <name> --from-literal=key=value
kubectl create configmap <name> --from-file=<path>
kubectl create configmap <name> --from-env-file=<path>

# Get ConfigMaps
kubectl get configmaps
kubectl get configmap <name> -o yaml
kubectl describe configmap <name>

# Update ConfigMap
kubectl edit configmap <name>
kubectl create configmap <name> --dry-run=client -o yaml | kubectl apply -f -

# Delete ConfigMap
kubectl delete configmap <name>

# Use in pod
kubectl set env pod/<pod-name> --from=configmap/<configmap-name>
```

---

## ✅ Clean Up

```bash
# Delete all resources created in this challenge
kubectl delete pod config-demo
kubectl delete pod config-demo-envfrom
kubectl delete pod config-demo-volume
kubectl delete pod config-demo-mixed
kubectl delete configmap app-config
kubectl delete configmap literal-config
kubectl delete configmap file-config
kubectl delete configmap dir-config
```

---

## ⭐ Support

If you found this challenge useful, please consider starring the repository and sharing it with your network!

---

Happy Kubernetes learning! 
