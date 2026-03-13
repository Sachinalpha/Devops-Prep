# Kubernetes Interview Preparation
## For DevOps Engineers (1 Year Experience)

---

## MOST ASKED INTERVIEW QUESTIONS WITH CODE

---

### Q1. What is Kubernetes?

```
Kubernetes (K8s) manages containers at scale

Without Kubernetes:
  run docker container manually
  if it crashes, restart manually
  scaling manually
  no load balancing

With Kubernetes:
  automatically restarts crashed containers
  auto scales based on load
  load balances traffic
  rolling updates with zero downtime
  self healing
```

---

### Q2. Kubernetes Architecture

```
Control Plane (Master):
  API Server      = entry point, all commands go here
  etcd            = database, stores all cluster state
  Scheduler       = decides which node to run pod on
  Controller      = watches state, makes sure desired state matches actual state

Worker Nodes:
  kubelet         = agent on each node, talks to control plane
  kube-proxy      = handles networking on node
  container runtime = runs containers (docker, containerd)

Objects:
  Pod             = smallest unit, one or more containers
  Deployment      = manages pods, handles updates
  Service         = exposes pods to network
  ConfigMap       = store non-secret configuration
  Secret          = store sensitive data
  Namespace       = logical separation of resources
  Ingress         = HTTP routing rules
  PersistentVolume = storage
```

---

### Q3. Create a basic Deployment

```yaml
# deployment.yaml

apiVersion: apps/v1          # api version for this resource type
kind: Deployment             # type of resource
metadata:
  name: myapp                # name of deployment
  namespace: default         # namespace (default if not specified)
  labels:
    app: myapp               # labels for identifying this deployment

spec:
  replicas: 3                # run 3 copies of the pod
  selector:
    matchLabels:
      app: myapp             # select pods with this label
  template:
    metadata:
      labels:
        app: myapp           # pod label (must match selector)
    spec:
      containers:
        - name: myapp                              # container name
          image: myacr.azurecr.io/myapp:1.0       # docker image
          ports:
            - containerPort: 3000                  # port app listens on
          resources:
            requests:                              # minimum resources needed
              memory: "64Mi"
              cpu: "250m"                          # 250 millicores = 0.25 CPU
            limits:                                # maximum resources allowed
              memory: "128Mi"
              cpu: "500m"
          env:
            - name: NODE_ENV
              value: "production"
```

```bash
kubectl apply -f deployment.yaml   # create or update deployment
kubectl get deployments            # list deployments
kubectl get pods                   # list pods
kubectl describe deployment myapp  # detailed info
kubectl delete deployment myapp    # delete deployment
```

---

### Q4. Create a Service

```yaml
# service.yaml
# Service exposes pods to network

apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: default

spec:
  selector:
    app: myapp           # route traffic to pods with this label
  ports:
    - protocol: TCP
      port: 80           # service port (external)
      targetPort: 3000   # container port (internal)
  type: LoadBalancer     # service type
  # ClusterIP   = only accessible inside cluster (default)
  # NodePort    = accessible on node IP and port
  # LoadBalancer = creates external load balancer (cloud)
```

```bash
kubectl apply -f service.yaml
kubectl get services
kubectl get svc                    # short form
kubectl describe service myapp-service
```

---

### Q5. Service types explained

```yaml
# ClusterIP - internal only
spec:
  type: ClusterIP              # default type
  # accessible only within cluster
  # use for internal communication between services

---

# NodePort - accessible on node IP
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 3000
      nodePort: 30080          # port on node (30000-32767)
  # access: http://node-ip:30080

---

# LoadBalancer - external access
spec:
  type: LoadBalancer
  # cloud creates external load balancer
  # get external IP: kubectl get svc
  # access: http://external-ip:80

---

# ExternalName - DNS alias
spec:
  type: ExternalName
  externalName: mydb.example.com
  # maps service to external DNS name
```

---

### Q6. ConfigMap - store configuration

```yaml
# configmap.yaml
# store non-sensitive configuration

apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  NODE_ENV: "production"         # key-value pairs
  PORT: "3000"
  DB_HOST: "mydb-service"
  app.properties: |              # multi-line value (file content)
    server.port=3000
    server.host=0.0.0.0
```

```yaml
# use configmap in deployment

spec:
  containers:
    - name: myapp
      image: myapp:1.0
      env:
        # method 1 - single value from configmap
        - name: NODE_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config      # configmap name
              key: NODE_ENV         # key in configmap

      envFrom:
        # method 2 - all values from configmap
        - configMapRef:
            name: app-config        # all keys become env variables

      volumeMounts:
        # method 3 - mount as file
        - name: config-volume
          mountPath: /app/config

  volumes:
    - name: config-volume
      configMap:
        name: app-config
        # app.properties key becomes /app/config/app.properties file
```

---

### Q7. Secrets - store sensitive data

```yaml
# secret.yaml
# base64 encoded values

apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: default
type: Opaque
data:
  DB_PASSWORD: bXlwYXNzd29yZA==    # base64 encoded "mypassword"
  API_KEY: c2VjcmV0a2V5MTIz        # base64 encoded value
```

```bash
# create base64 value
echo -n "mypassword" | base64     # outputs bXlwYXNzd29yZA==

# create secret directly from command
kubectl create secret generic app-secret \
  --from-literal=DB_PASSWORD=mypassword \
  --from-literal=API_KEY=secretkey123

# create secret from file
kubectl create secret generic app-secret \
  --from-file=./secrets.env
```

```yaml
# use secret in deployment
spec:
  containers:
    - name: myapp
      image: myapp:1.0
      env:
        # single value from secret
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret      # secret name
              key: DB_PASSWORD      # key in secret

      envFrom:
        # all values from secret
        - secretRef:
            name: app-secret
```

---

### Q8. Namespace

```yaml
# namespace.yaml
# logical separation of resources

apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

```bash
kubectl create namespace dev
kubectl create namespace prod

# create resource in specific namespace
kubectl apply -f deployment.yaml -n dev
kubectl apply -f deployment.yaml -n prod

# list resources in namespace
kubectl get pods -n dev
kubectl get all -n dev             # list everything in namespace
kubectl get pods --all-namespaces  # list pods in all namespaces

# set default namespace
kubectl config set-context --current --namespace=dev
```

---

### Q9. Rolling update and rollback

```bash
# update image version
kubectl set image deployment/myapp myapp=myapp:2.0
# kubernetes does rolling update automatically
# old pods removed one by one
# new pods added one by one
# zero downtime

# check rollout status
kubectl rollout status deployment/myapp

# check rollout history
kubectl rollout history deployment/myapp

# rollback to previous version
kubectl rollout undo deployment/myapp

# rollback to specific version
kubectl rollout undo deployment/myapp --to-revision=2
```

```yaml
# control rolling update in deployment
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # max extra pods during update
      maxUnavailable: 0    # max pods that can be unavailable
  # ensures zero downtime
```

---

### Q10. Resource requests and limits

```yaml
spec:
  containers:
    - name: myapp
      image: myapp:1.0
      resources:
        requests:
          memory: "64Mi"     # Mi = Mebibytes
          cpu: "250m"        # m = millicores, 1000m = 1 CPU core
        limits:
          memory: "128Mi"
          cpu: "500m"

# requests = guaranteed resources
# limits = maximum allowed resources
# if container exceeds memory limit = OOMKilled (Out of Memory)
# if container exceeds CPU limit = throttled (not killed)

# CPU units:
# 1 CPU = 1000m
# 0.5 CPU = 500m
# 0.25 CPU = 250m

# Memory units:
# Ki = Kibibytes (1024 bytes)
# Mi = Mebibytes (1024 Ki)
# Gi = Gibibytes (1024 Mi)
```

---

### Q11. Liveness and Readiness probes

```yaml
spec:
  containers:
    - name: myapp
      image: myapp:1.0
      livenessProbe:
        # is container alive? if fails, restart container
        httpGet:
          path: /health      # endpoint to check
          port: 3000
        initialDelaySeconds: 30    # wait 30s before first check
        periodSeconds: 10          # check every 10 seconds
        failureThreshold: 3        # restart after 3 failures

      readinessProbe:
        # is container ready to receive traffic?
        # if fails, remove from service endpoints (no traffic)
        # does NOT restart container
        httpGet:
          path: /ready
          port: 3000
        initialDelaySeconds: 10
        periodSeconds: 5
        failureThreshold: 3

# liveness  = is it running?  → restart if fails
# readiness = is it ready?    → remove from load balancer if fails
```

---

### Q12. Horizontal Pod Autoscaler (HPA)

```yaml
# hpa.yaml
# auto scale pods based on CPU/memory usage

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp                 # deployment to scale
  minReplicas: 2                # minimum pods
  maxReplicas: 10               # maximum pods
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70   # scale when CPU > 70%
```

```bash
kubectl apply -f hpa.yaml
kubectl get hpa                  # check autoscaler status
```

---

### Q13. Most used kubectl commands

```bash
# basic
kubectl get pods                          # list pods
kubectl get pods -o wide                  # list pods with node info
kubectl get pods -n namespace             # list pods in namespace
kubectl describe pod pod-name             # detailed pod info
kubectl logs pod-name                     # view pod logs
kubectl logs -f pod-name                  # follow logs
kubectl exec -it pod-name -- bash         # open shell in pod
kubectl delete pod pod-name               # delete pod

# deployments
kubectl get deployments
kubectl scale deployment myapp --replicas=5
kubectl set image deployment/myapp myapp=myapp:2.0
kubectl rollout status deployment/myapp
kubectl rollout undo deployment/myapp

# services
kubectl get services
kubectl get svc

# config
kubectl get configmaps
kubectl get secrets

# apply/delete
kubectl apply -f file.yaml
kubectl delete -f file.yaml
kubectl apply -f ./           # apply all yaml files in folder

# cluster info
kubectl cluster-info
kubectl get nodes
kubectl top nodes              # resource usage of nodes
kubectl top pods               # resource usage of pods
```

---

### Q14. Ingress - HTTP routing

```yaml
# ingress.yaml
# route HTTP traffic to different services based on path or host

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /

spec:
  rules:
    - host: myapp.example.com              # domain name
      http:
        paths:
          - path: /api                     # route /api to api service
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /                        # route / to frontend service
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

---

### Q15. PersistentVolume and PersistentVolumeClaim

```yaml
# persistentvolumeclaim.yaml
# request storage

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myapp-pvc
spec:
  accessModes:
    - ReadWriteOnce          # one node can read/write at a time
  resources:
    requests:
      storage: 5Gi           # request 5GB storage
  storageClassName: default  # storage class (cloud provider specific)
```

```yaml
# use PVC in deployment
spec:
  containers:
    - name: myapp
      image: myapp:1.0
      volumeMounts:
        - name: mydata
          mountPath: /app/data    # mount point in container
  volumes:
    - name: mydata
      persistentVolumeClaim:
        claimName: myapp-pvc      # PVC name
```

---

### Common Mistakes to Avoid

```
1. Not setting resource requests and limits
2. Not using readiness probes (traffic to unready pods)
3. Using latest image tag (unpredictable deployments)
4. Storing secrets in ConfigMap instead of Secret
5. Not using namespaces to separate environments
6. Not understanding difference between liveness and readiness probe
7. Not setting pod disruption budget for production
8. Forgetting to update image tag in deployment
```
