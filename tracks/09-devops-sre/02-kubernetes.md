# 02. Kubernetes

[← Назад к списку тем](README.md)

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Kubernetes Architecture                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Control Plane                          Worker Nodes                │
│  ┌──────────────────────┐              ┌──────────────────────┐    │
│  │ API Server           │◄─────────────┤ kubelet              │    │
│  │ (kube-apiserver)     │              │ (node agent)         │    │
│  ├──────────────────────┤              ├──────────────────────┤    │
│  │ etcd                 │              │ kube-proxy           │    │
│  │ (distributed KV)     │              │ (network proxy)      │    │
│  ├──────────────────────┤              ├──────────────────────┤    │
│  │ Scheduler            │              │ Container Runtime    │    │
│  │ (kube-scheduler)     │              │ (containerd/CRI-O)   │    │
│  ├──────────────────────┤              └──────────────────────┘    │
│  │ Controller Manager   │                                          │
│  │ (kube-controller-mgr)│                                          │
│  └──────────────────────┘                                          │
│                                                                     │
│  Control Plane: manages cluster state                               │
│  Worker Nodes: run actual workloads                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Components

```
┌─────────────────┬──────────────────────────────────────────────────┐
│ Component       │ Purpose                                          │
├─────────────────┼──────────────────────────────────────────────────┤
│ API Server      │ Frontend for control plane, all communication    │
│ etcd            │ Distributed key-value store for cluster state    │
│ Scheduler       │ Assigns pods to nodes                            │
│ Controller Mgr  │ Runs controllers (Deployment, ReplicaSet, etc)   │
│ kubelet         │ Agent on each node, manages pods                 │
│ kube-proxy      │ Network proxy, implements Services               │
└─────────────────┴──────────────────────────────────────────────────┘
```

---

## Core Resources

### Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
    livenessProbe:
      httpGet:
        path: /health
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 5
```

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

### Service

```yaml
# ClusterIP (default) - internal only
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP

---
# NodePort - external via node ports
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080

---
# LoadBalancer - cloud load balancer
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

### ConfigMap & Secret

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "postgres"
  LOG_LEVEL: "info"
  config.json: |
    {
      "feature_flags": {
        "new_ui": true
      }
    }

---
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  DATABASE_PASSWORD: "secretpassword"
  API_KEY: "myapikey"
```

### Using in Pod

```yaml
spec:
  containers:
  - name: app
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DATABASE_HOST
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: DATABASE_PASSWORD
    volumeMounts:
    - name: config
      mountPath: /etc/config
  volumes:
  - name: config
    configMap:
      name: app-config
```

---

## Networking

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Kubernetes Networking                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Pod-to-Pod: Every pod can communicate with every other pod         │
│              using pod IPs (flat network)                           │
│                                                                     │
│  Pod-to-Service: Services provide stable IP/DNS for pod groups      │
│                                                                     │
│  External-to-Service:                                               │
│  - NodePort: Expose on node IP:port                                 │
│  - LoadBalancer: Cloud load balancer                                │
│  - Ingress: HTTP(S) routing                                         │
│                                                                     │
│  DNS:                                                               │
│  - Service: <service>.<namespace>.svc.cluster.local                 │
│  - Pod: <pod-ip>.<namespace>.pod.cluster.local                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls
```

---

## Storage

```yaml
# PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard

---
# StatefulSet with PVC
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

---

## Scaling

### Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

---

## kubectl Commands

```bash
# Basic operations
kubectl get pods                    # List pods
kubectl get pods -o wide            # More details
kubectl get all                     # All resources
kubectl get pods -A                 # All namespaces

# Create/Apply
kubectl apply -f manifest.yaml      # Create/update
kubectl delete -f manifest.yaml     # Delete

# Debugging
kubectl describe pod mypod          # Detailed pod info
kubectl logs mypod                  # Pod logs
kubectl logs -f mypod               # Follow logs
kubectl logs mypod -c container     # Specific container
kubectl exec -it mypod -- bash      # Shell into pod

# Port forwarding
kubectl port-forward pod/mypod 8080:80
kubectl port-forward svc/mysvc 8080:80

# Scaling
kubectl scale deployment myapp --replicas=5

# Rollout
kubectl rollout status deployment/myapp
kubectl rollout history deployment/myapp
kubectl rollout undo deployment/myapp
kubectl rollout restart deployment/myapp

# Context
kubectl config get-contexts         # List contexts
kubectl config use-context prod     # Switch context
kubectl config current-context      # Current context
```

---

## Health Checks

```yaml
spec:
  containers:
  - name: app
    # Liveness: Is the container healthy?
    # Fails → container restarts
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 10
      failureThreshold: 3

    # Readiness: Can the container receive traffic?
    # Fails → removed from Service endpoints
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5

    # Startup: For slow-starting containers
    # Fails → liveness/readiness don't run yet
    startupProbe:
      httpGet:
        path: /startup
        port: 8080
      failureThreshold: 30
      periodSeconds: 10
```

---

## Resource Management

```yaml
spec:
  containers:
  - name: app
    resources:
      # Requests: Guaranteed resources
      # Used for scheduling decisions
      requests:
        memory: "256Mi"
        cpu: "250m"        # 250 millicores = 0.25 CPU

      # Limits: Maximum allowed
      # Exceeding memory = OOM killed
      # Exceeding CPU = throttled
      limits:
        memory: "512Mi"
        cpu: "500m"
```

### QoS Classes

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Quality of Service                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Guaranteed: requests == limits (for all containers)                │
│  - Highest priority, last to be evicted                             │
│                                                                     │
│  Burstable: requests < limits                                       │
│  - Medium priority                                                  │
│                                                                     │
│  BestEffort: no requests or limits                                  │
│  - Lowest priority, first to be evicted                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Deployments Strategies

```yaml
# Rolling Update (default)
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # Extra pods during update
      maxUnavailable: 0    # Always have min replicas

# Recreate (downtime)
spec:
  strategy:
    type: Recreate
```

### Blue-Green / Canary

```yaml
# Blue-Green: Switch service selector
# Canary: Use multiple deployments with weights (Istio, etc.)

# Simple canary with multiple deployments
# Main deployment: 90% traffic
# Canary deployment: 10% traffic
# Adjust replicas ratio
```

---

## RBAC

```yaml
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa

---
# Role (namespace-scoped)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]

---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
subjects:
- kind: ServiceAccount
  name: app-sa
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## См. также

- [Containers & Docker](./01-containers-docker.md) — основы контейнеризации, на которых построен Kubernetes
- [CI/CD](./03-ci-cd.md) — автоматизация деплоя приложений в Kubernetes

---

## На интервью

### Типичные вопросы

1. **Pod vs Deployment?**
   - Pod: smallest deployable unit, one or more containers
   - Deployment: manages ReplicaSets, provides updates, scaling
   - Don't create pods directly in production

2. **Service types?**
   - ClusterIP: internal only
   - NodePort: expose on node port
   - LoadBalancer: cloud LB
   - ExternalName: DNS alias

3. **Liveness vs Readiness?**
   - Liveness: is container healthy? Fails → restart
   - Readiness: can receive traffic? Fails → no traffic
   - Startup: for slow-starting apps

4. **How does rolling update work?**
   - Creates new ReplicaSet
   - Gradually scales up new, scales down old
   - maxSurge/maxUnavailable control the rate

5. **StatefulSet vs Deployment?**
   - StatefulSet: stable network identity, ordered scaling, persistent storage
   - Use for databases, distributed systems
   - Deployment: stateless apps, interchangeable pods

6. **Resource requests vs limits?**
   - Requests: guaranteed, used for scheduling
   - Limits: maximum allowed
   - Set both for predictable behavior

---

[← Назад к списку тем](README.md)
