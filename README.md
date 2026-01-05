# Kubernetes fundamentals

## Table of Contents
1. [Prerequisites & Setup](#prerequisites--setup)
2. [Docker → Kubernetes Concept Map](#docker--kubernetes-concept-map)
3. [Learning Path Overview](#learning-path-overview)
4. [Day 1: First Deployment](#day-1-first-deployment)
5. [Day 2: Configuration Management](#day-2-configuration-management)
6. [Day 3: Persistent Storage](#day-3-persistent-storage)
7. [Day 4: Scheduled Jobs](#day-4-scheduled-jobs)
8. [Day 5: Production Patterns](#day-5-production-patterns)
9. [Essential Commands Cheat Sheet](#essential-commands-cheat-sheet)
10. [Troubleshooting Guide](#troubleshooting-guide)

---

## Prerequisites & Setup

### Local Development

#### Option 1: Docker Desktop (Windows/Mac)
**Cost**: Free for personal use  
**What it includes**: Docker + built-in Kubernetes cluster

**Setup**:
1. Install Docker Desktop: https://www.docker.com/products/docker-desktop
2. Open Docker Desktop → Settings → Kubernetes
3. Check "Enable Kubernetes" → Apply & Restart
4. Verify: `kubectl version --client` (should show version info)

**Pros**: One app, GUI management, seamless with existing Docker workflow  
**Cons**: Uses 2-4GB RAM, Windows/Mac only

#### Option 2: Minikube (Cross-platform)
**Cost**: Free  
**What it includes**: Single-node Kubernetes cluster in VM/container

**Setup**:
```bash
# Install kubectl (Kubernetes CLI)
# Windows (via Chocolatey):
choco install kubernetes-cli

# Mac (via Homebrew):
brew install kubectl

# Linux:
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install Minikube
# Windows:
choco install minikube

# Mac:
brew install minikube

# Linux:
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start cluster
minikube start --driver=docker  # or virtualbox, hyperv
```

**Pros**: Lightweight, works everywhere, perfect for learning  
**Cons**: Requires separate install from Docker

### Recommended for This Guide
**Docker Desktop** if you're on Windows/Mac (simplest path)  
**Minikube** if you're on Linux or want lightweight option

### Verify Installation
```bash
# Check kubectl is installed
kubectl version --client

# Check cluster is running
kubectl cluster-info

# Expected output:
# Kubernetes control plane is running at https://...

# Create a test deployment
kubectl create deployment hello --image=nginx:alpine

# Verify it's running
kubectl get pods
# Should show: hello-xxxxx   1/1   Running

# Expose it
kubectl expose deployment hello --port=80 --type=LoadBalancer

# Access it
curl http://localhost
# Should show: "Welcome to nginx!"

# Clean up
kubectl delete deployment hello
kubectl delete service hello
```

---

## Practice
We'll deploy a simple .NET API + PostgreSQL database across all exercises.

---

## Exercise 1: First Deployment

### Goal
Deploy a stateless web API with 3 replicas, accessible via LoadBalancer.

### Docker Equivalent
```bash
docker run -d -p 8080:80 --name api --restart=always myapp:latest
docker run -d -p 8080:80 --name api2 --restart=always myapp:latest
docker run -d -p 8080:80 --name api3 --restart=always myapp:latest
```

### Kubernetes Approach
**Single file** defines desired state: "I want 3 replicas always running"

#### Step 1.1: Create Deployment YAML

Create `exercise1-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapi
  labels:
    app: myapi
spec:
  replicas: 3  # "I want 3 copies running"
  selector:
    matchLabels:
      app: myapi  # "Manage pods with this label"
  template:
    metadata:
      labels:
        app: myapi  # Label for pods
    spec:
      containers:
      - name: api
        image: mcr.microsoft.com/dotnet/samples:aspnetapp  # Public demo image
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
```

**Key Concepts**:
- `replicas: 3`: Kubernetes maintains exactly 3 pods running
- `selector.matchLabels`: How Deployment finds its pods
- `template.spec.containers`: Pod definition (like Dockerfile)
- `resources`: CPU/memory allocation (prevents one app consuming all resources)

#### Step 1.2: Create Service YAML

Create `exercise1-service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapi-service
spec:
  type: LoadBalancer  # Exposes externally (like -p 8080:80)
  selector:
    app: myapi  # Routes traffic to pods with this label
  ports:
  - port: 80        # External port
    targetPort: 8080  # Container port
```

**Key Concepts**:
- **Service Types**:
  - `ClusterIP` (default): Internal only (like Docker network)
  - `LoadBalancer`: External access (like `-p` port mapping)
  - `NodePort`: Exposes on each node's IP
- `selector`: Must match Deployment's pod labels

### Kubernetes mental model

Deployment = "What to run"

- Defines pod configuration (image, resources, env vars)
- Manages desired state (replicas: 3 = "keep 3 pods running")
- Handles lifecycle (restarts, updates, rollbacks)

Service = "How to make them interact"

- Provides stable network endpoint (DNS name, IP)
- Load balances traffic across pods
- Defines access type (internal/external)

#### Why separate?

You can have:

- 1 Deployment with multiple Services (internal + external access)
- Multiple Deployments sharing 1 Service (A/B testing, blue-green)

**This separation is what makes Kubernetes powerful - you change networking without touching application config**.

```yaml
# Same Deployment, different Service types for different use cases:

# Internal only (microservice communication)
Service type: ClusterIP → postgres.default.svc.cluster.local:5432

# External access (public API)
Service type: LoadBalancer → http://my-api.example.com
```

#### Step 1.3: Deploy to Cluster

```bash
# Apply both files
kubectl apply -f exercise1-deployment.yaml
kubectl apply -f exercise1-service.yaml

# Verify deployment
kubectl get deployments
# NAME    READY   UP-TO-DATE   AVAILABLE   AGE
# myapi   3/3     3            3           30s

# Verify pods
kubectl get pods
# NAME                     READY   STATUS    RESTARTS   AGE
# myapi-7f8d9c-abcd        1/1     Running   0          30s
# myapi-7f8d9c-efgh        1/1     Running   0          30s
# myapi-7f8d9c-ijkl        1/1     Running   0          30s

# Verify service
kubectl get services
# NAME            TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
# myapi-service   LoadBalancer   10.96.123.45    localhost     80:30123/TCP   30s
```

#### Step 1.4: Access Your Application

**Docker Desktop**:
```bash
# Service is available at localhost
curl http://localhost
```

**Minikube**:
```bash
# Get external URL
minikube service myapi-service --url
# Output: http://192.168.49.2:30123

# Open in browser
minikube service myapi-service
```

#### Step 1.5: Test Self-Healing

```bash
# Delete one pod
kubectl delete pod myapi-7f8d9c-abcd

# Immediately check pods
kubectl get pods
# Kubernetes automatically creates a new pod to maintain replicas: 3

# Watch real-time
kubectl get pods --watch
```

### Validation Checklist
- [X] 3 pods running (`kubectl get pods`)
- [X] Service shows EXTERNAL-IP (`kubectl get svc`)
- [X] Application accessible via browser/curl
- [X] Deleting a pod creates a new one automatically

### Common Issues

**Issue**: Pods stuck in `Pending` state  
**Solution**: Check resources
```bash
kubectl describe pod myapi-7f8d9c-abcd
# Look for "Insufficient cpu" or "Insufficient memory"
```

**Issue**: Service EXTERNAL-IP shows `<pending>` forever  
**Solution**: 
- **Docker Desktop**: Check Kubernetes is enabled in settings
- **Minikube**: Use `minikube service myapi-service` instead
- **Kind**: Use port-forward: `kubectl port-forward svc/myapi-service 8080:80`

**Issue**: `error: the server doesn't have a resource type "deployments"`  
**Solution**: Check cluster is running: `kubectl cluster-info`

### Key Takeaways
- Deployments manage desired state (replicas)
- Services provide stable network endpoints
- Kubernetes automatically heals failures
- `kubectl apply -f` is idempotent (safe to run multiple times)

---

## Exercise 2: Configuration Management

### Goal
Separate configuration from code using ConfigMaps and Secrets.

### Docker Equivalent
```bash
docker run -d \
  -e DB_HOST=postgres \
  -e DB_PASSWORD=secret123 \
  -e LOG_LEVEL=Debug \
  myapp:latest
```

### Kubernetes Approach
**ConfigMap** (non-sensitive) + **Secret** (sensitive) → injected as env vars or files

#### Step 2.1: Create ConfigMap

Create `exercise2-configmap.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # Key-value pairs
  LOG_LEVEL: "Information"
  FEATURE_FLAG_NEWUI: "true"
  
  # Or entire files
  appsettings.json: |
    {
      "Logging": {
        "LogLevel": {
          "Default": "Information"
        }
      },
      "AllowedHosts": "*"
    }
```

#### Step 2.2: Create Secret

**Option A: From Command Line** (recommended for sensitive data)
```bash
kubectl create secret generic db-credentials \
  --from-literal=password='MySecureP@ssw0rd!' \
  --from-literal=connection-string='Server=postgres;Database=mydb;User Id=admin;Password=MySecureP@ssw0rd!'
```

**Option B: From YAML** (base64 encoded)
Create `day2-secret.yaml`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  # Values must be base64 encoded
  password: TXlTZWN1cmVQQHNzdzByZCE=
  connection-string: U2VydmVyPXBvc3RncmVzO0RhdGFiYXNlPW15ZGI7VXNlciBJZD1hZG1pbjtQYXNzd29yZD1NeVNlY3VyZVBAc3N3MHJkIQ==
```

**Encoding secrets**:
```bash
# Encode
echo -n 'MySecureP@ssw0rd!' | base64
# Output: TXlTZWN1cmVQQHNzdzByZCE=

# Decode (verify)
echo 'TXlTZWN1cmVQQHNzdzByZCE=' | base64 --decode
```

#### Step 2.3: Update Deployment to Use Config

Create `exercise2-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapi
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapi
  template:
    metadata:
      labels:
        app: myapi
    spec:
      containers:
      - name: api
        image: mcr.microsoft.com/dotnet/samples:aspnetapp
        ports:
        - containerPort: 8080
        
        # Environment variables from ConfigMap
        env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL
        
        - name: FEATURE_FLAG_NEWUI
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: FEATURE_FLAG_NEWUI
        
        # Environment variables from Secret
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        
        - name: CONNECTION_STRING
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: connection-string
        
        # Mount entire ConfigMap as file
        volumeMounts:
        - name: config-volume
          mountPath: /app/config
          readOnly: true
      
      volumes:
      - name: config-volume
        configMap:
          name: app-config
```

#### Step 2.4: Deploy and Verify

```bash
# Create ConfigMap
kubectl apply -f exercise2-configmap.yaml

# Create Secret (if using YAML)
kubectl apply -f exercise2-secret.yaml
# Or use command line version from Step 2.2

# Update Deployment
kubectl apply -f exercise2-deployment.yaml

# Verify environment variables in pod
kubectl get pods  # Get pod name
kubectl exec myapi-7f8d9c-abcd -- env | grep -E 'LOG_LEVEL|DB_PASSWORD'
# Should show LOG_LEVEL=Information

# Verify mounted file
kubectl exec myapi-7f8d9c-abcd -- cat /app/config/appsettings.json
```

#### Step 2.5: Update Config Without Redeploying

```bash
# Edit ConfigMap
kubectl edit configmap app-config
# Change LOG_LEVEL to "Debug"

# Restart pods to pick up new config
kubectl rollout restart deployment myapi

# Verify change
kubectl exec myapi-NEW_POD_NAME -- env | grep LOG_LEVEL
```

### Validation Checklist
- [X] ConfigMap created (`kubectl get configmap`)
- [X] Secret created (`kubectl get secret`)
- [X] Environment variables visible in pod
- [X] File mounted at `/app/config/appsettings.json`
- [X] Config change triggers restart

### Common Issues

**Issue**: Secret values show as base64 in pod  
**Solution**: This is correct. Applications decode automatically.

**Issue**: Pod won't start after adding config  
**Solution**: Check references match
```bash
kubectl describe pod myapi-7f8d9c-abcd
# Look for "configmap "app-config" not found"
```

**Issue**: Changes to ConfigMap don't reflect in pod  
**Solution**: ConfigMaps don't auto-reload. Use `kubectl rollout restart`

### Key Takeaways
- ConfigMaps for non-sensitive configuration
- Secrets for passwords, tokens, certificates (base64 encoded)
- Inject as env vars OR mount as files
- Changing ConfigMap/Secret requires pod restart

---

## Exercise 3: Persistent Storage

### Goal
Deploy PostgreSQL with persistent storage that survives pod restarts.

### Docker Equivalent
```bash
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=mysecret \
  -v pgdata:/var/lib/postgresql/data \
  postgres:15

docker volume create pgdata
```

### Kubernetes Approach
**StatefulSet** (stable pod names) + **PersistentVolumeClaim** (storage request)

#### Step 3.1: Create Storage Class (Check Default)

```bash
# Check available storage classes
kubectl get storageclass

# Docker Desktop/Minikube provide default storage
# NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE
# standard (default)   k8s.io/minikube-hostpath   Delete          Immediate
```

You typically don't need to create this manually.

#### Step 3.2: Create StatefulSet with PVC

Create `exercise3-postgres.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None  # Headless service (no load balancing needed)
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres  # Must match Service name above
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
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          value: mysecretpassword  # In production, use Secret
        - name: POSTGRES_DB
          value: myappdb
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
          subPath: postgres  # Avoid permission issues
  
  # PersistentVolumeClaim template (created per replica)
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

**Key Differences from Deployment**:
- **StatefulSet**: Pods get stable names (`postgres-0`, not `postgres-7f8d9c-abcd`)
- **Headless Service**: Direct pod DNS (`postgres-0.postgres.default.svc.cluster.local`)
- **volumeClaimTemplates**: Persistent storage per pod

#### Step 3.3: Deploy and Verify Storage

```bash
# Create StatefulSet
kubectl apply -f exercise3-postgres.yaml

# Check pod
kubectl get pods
# NAME         READY   STATUS    RESTARTS   AGE
# postgres-0   1/1     Running   0          30s

# Check PersistentVolumeClaim
kubectl get pvc
# NAME                        STATUS   VOLUME                                     CAPACITY   ACCESS MODES
# postgres-storage-postgres-0 Bound    pvc-abc123                                 1Gi        RWO

# Check PersistentVolume (auto-created)
kubectl get pv
```

#### Step 3.4: Test Data Persistence

```bash
# Connect to database
kubectl exec -it postgres-0 -- psql -U postgres -d myappdb

# Create table and insert data
CREATE TABLE test (id SERIAL PRIMARY KEY, value TEXT);
INSERT INTO test (value) VALUES ('Hello Kubernetes!');
SELECT * FROM test;
# Should show: 1 | Hello Kubernetes!
\q

# Delete the pod (simulate crash)
kubectl delete pod postgres-0

# Wait for recreation
kubectl wait --for=condition=Ready pod/postgres-0 --timeout=60s

# Verify data survived
kubectl exec -it postgres-0 -- psql -U postgres -d myappdb -c "SELECT * FROM test;"
# Data still exists!
```

#### Step 3.5: Connect Application to Database

Duplicate `exercise2-deployment.yaml` and amend to `exercise3-deployment.yaml` to add database connection:
```yaml
# Add to containers.env section:
- name: DB_HOST
  value: postgres.default.svc.cluster.local
- name: DB_PORT
  value: "5432"
- name: DB_NAME
  value: myappdb
- name: DB_USER
  value: postgres
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: password
```

#### Step 3.6: Job to Test Database Connection

```bash
# 1. Create test table in database
kubectl --namespace kubernetes-fundamentals exec -it postgres-0 -- psql -U postgres -d myappdb
CREATE TABLE connection_test (id SERIAL PRIMARY KEY, pod_name VARCHAR(255), connected_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP);
```

Deploy `exercise3-job.yaml` job to write data from pod to database:
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-connection-test
  namespace: kubernetes-fundamentals
spec:
  template:
    spec:
      containers:
      - name: test
        image: postgres:15
        command:
        - /bin/bash
        - -c
        - |
          echo "Testing connection to database..."
          if psql -h postgres.kubernetes-fundamentals.svc.cluster.local -U postgres -d myappdb -c "INSERT INTO connection_test (pod_name) VALUES ('test-job-$(hostname)');" ; then
            echo "✓ Successfully wrote to database"
            psql -h postgres.kubernetes-fundamentals.svc.cluster.local -U postgres -d myappdb -c "SELECT * FROM connection_test;"
          else
            echo "✗ Database connection failed"
            exit 1
          fi
        env:
        - name: PGPASSWORD
          value: mysecretpassword
      restartPolicy: Never
```

```terminal
Testing connection to database...
INSERT 0 1
✓ Successfully wrote to database
 id |             pod_name              |        connected_at
----+-----------------------------------+----------------------------
  1 | test-job-db-connection-test-f5w5m | 2026-01-05 13:59:00.118148
(1 row)
```

#### Step 3.7: Safe Volume Deletion

```bash
# 1. Backup data first (CRITICAL)
kubectl exec postgres-0 -- pg_dump -U postgres myappdb > backup-$(date +%Y%m%d).sql

# Verify backup exists
ls -lh backup-*.sql

# 2. Delete StatefulSet (stops application)
kubectl delete statefulset postgres

# 3. Delete Service (removes network endpoint)
kubectl delete service postgres

# 4. Delete PVC (removes storage claim)
kubectl delete pvc postgres-storage-postgres-0

# 5. Verify PV is deleted (automatic with default ReclaimPolicy)
kubectl get pv
# Should show no volumes, or previous volume in "Released" state
```

##### Production Deletion

```bash
# 1. Scale down StatefulSet (graceful shutdown)
kubectl scale statefulset postgres --replicas=0

# 2. Wait for shutdown
kubectl wait --for=delete pod/postgres-0 --timeout=60s

# 3. Backup
kubectl exec postgres-0 -- pg_dumpall -U postgres > full-backup.sql

# 4. Delete resources
kubectl delete statefulset postgres
kubectl delete service postgres
kubectl delete pvc postgres-storage-postgres-0
```

### Validation Checklist
- [X] PVC created and bound (`kubectl get pvc`)
- [X] PV created automatically (`kubectl get pv`)
- [X] PostgreSQL pod running (`kubectl get pods`)
- [X] Data survives pod deletion
- [X] Application can connect to database
- [X] Application writes data to database successfully

### Common Issues

**Issue**: PVC stuck in `Pending` state  
**Solution**: Check storage class exists
```bash
kubectl describe pvc postgres-storage-postgres-0
# Look for "no persistent volumes available"
```

**Issue**: Permission denied on `/var/lib/postgresql/data`  
**Solution**: Use `subPath: postgres` in volumeMount (already in example)

**Issue**: Can't connect to database from application  
**Solution**: Use full DNS name: `postgres.default.svc.cluster.local`

### Key Takeaways
- **StatefulSet**: For stateful apps (databases, message queues)
- **Deployment**: For stateless apps (web APIs, workers)
- PVC requests storage; PV provides storage
- Data persists across pod restarts/deletions
- Headless Service provides stable DNS names

---

## Exercise 4: Scheduled Jobs

### Goal
Run a nightly database backup job at 2 AM daily.

### Docker Equivalent
```bash
# Cron entry
0 2 * * * docker run --rm backup-script:latest

# One-time job
docker run --rm migration:latest
```

### Kubernetes Approach
**CronJob** (scheduled) + **Job** (one-time)

#### Step 4.1: Create CronJob for Backups

Create test table in database using `exercise3-postgres.yaml`:
```bash
kubectl apply -f exercise3-postgres.yaml
kubectl --namespace kubernetes-fundamentals exec -it postgres-0 -- psql -U postgres -d myappdb
CREATE TABLE test (id SERIAL PRIMARY KEY, job_name VARCHAR(255), connected_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP);
```

Create `exercise4-cronjob.yaml`:
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
  namespace: kubernetes-fundamentals
spec:
  schedule: "*/1 * * * *"  # Cron syntax: minute hour day month weekday
  successfulJobsHistoryLimit: 3  # Keep last 3 successful jobs
  failedJobsHistoryLimit: 1      # Keep last 1 failed job
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:15
            command:
            - /bin/bash
            - -c
            - |
              echo "Testing writing to database..."
              if psql -h postgres.kubernetes-fundamentals.svc.cluster.local -U postgres -d myappdb -c "INSERT INTO test (job_name) VALUES ('cron-job-$(hostname)');" ; then
                echo "✓ Successfully wrote to database"
                psql -h postgres.kubernetes-fundamentals.svc.cluster.local -U postgres -d myappdb -c "SELECT * FROM test;"
              else
                echo "✗ Database connection failed"
                exit 1
              fi
            env:
            - name: PGPASSWORD
              value: mysecretpassword
          
          restartPolicy: OnFailure  # Retry on failure
```

#### Step 4.2: Deploy and Test Immediately

```bash
# Create CronJob with reduced schedule for testing (every minute)
kubectl apply -f exercise4-cronjob.yaml

# View logs
kubectl logs --namespace kubernetes-fundamentals db-backup-<pod_name>

Testing writing to database...
INSERT 0 1
✓ Successfully wrote to database
 id |             job_name              |        connected_at
----+-----------------------------------+----------------------------
  1 | cron-job-db-backup-29460445-8s7l2 | 2026-01-05 15:25:00.738352

# Check if row exists in database
kubectl --namespace kubernetes-fundamentals exec -it postgres-0 -- psql -U postgres -d myappdb
myappdb=# SELECT * FROM test;
 id |             job_name              |        connected_at
----+-----------------------------------+----------------------------
  1 | cron-job-db-backup-29460445-8s7l2 | 2026-01-05 15:25:00.738352
  2 | cron-job-db-backup-29460446-ss9tc | 2026-01-05 15:26:00.768525
  3 | cron-job-db-backup-29460447-c8ts8 | 2026-01-05 15:27:00.76739
  4 | cron-job-db-backup-29460448-txmn4 | 2026-01-05 15:28:00.757523
  5 | cron-job-db-backup-29460449-8hv4g | 2026-01-05 15:29:00.736292
  6 | cron-job-db-backup-29460450-gnnx8 | 2026-01-05 15:30:00.761418
(6 rows)
```

Common cron schedules:
```yaml
# Every 15 minutes
schedule: "*/15 * * * *"

# Every hour at minute 30
schedule: "30 * * * *"

# Every day at 3:30 AM
schedule: "30 3 * * *"

# Every Monday at 9 AM
schedule: "0 9 * * 1"

# First day of every month at midnight
schedule: "0 0 1 * *"
```

### Validation Checklist
- [X] CronJob created (`kubectl get cronjobs`)
- [X] Cron job completes successfully
- [X] Rows insered in database table

### Common Issues

**Issue**: Job fails with `ImagePullBackOff`  
**Solution**: Verify image exists and is accessible
```bash
kubectl describe job db-backup-manual
```

**Issue**: Job never completes (stuck in Running)  
**Solution**: Check logs for errors
```bash
kubectl logs job/db-backup-manual --follow
```

**Issue**: CronJob not triggering  
**Solution**: Check schedule syntax and timezone (UTC by default)
```bash
kubectl get cronjobs
# Look at LAST SCHEDULE column
```

### Key Takeaways
- **CronJob**: Scheduled recurring tasks
- **Job**: One-time task with completion guarantee
- `restartPolicy: OnFailure` vs `Never`
- Manual trigger: `kubectl create job --from=cronjob/name`
- Job history preserved per `successfulJobsHistoryLimit`

---

## Exercise 5: Production Patterns

### Goal
Implement health checks, resource limits, and organized configuration.

### Production-Ready Deployment Template

Create `exercise5-production-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapi
  labels:
    app: myapi
    version: v1.2.0
spec:
  replicas: 3
  
  # Rolling update strategy
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Create 1 extra pod during update
      maxUnavailable: 0  # Keep all pods available during update
  
  selector:
    matchLabels:
      app: myapi
  
  template:
    metadata:
      labels:
        app: myapi
        version: v1.2.0
    spec:
      containers:
      - name: api
        image: mcr.microsoft.com/dotnet/samples:aspnetapp
        imagePullPolicy: Always  # Always pull latest image
        
        ports:
        - name: http
          containerPort: 8080
        
        # Resource limits (CRITICAL for production)
        resources:
          requests:
            memory: "128Mi"  # Guaranteed allocation
            cpu: "250m"      # 0.25 CPU cores
          limits:
            memory: "256Mi"  # Maximum allowed
            cpu: "500m"      # 0.5 CPU cores
        
        # Liveness probe (restart if unhealthy)
        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 30  # Wait 30s before first check
          periodSeconds: 10        # Check every 10s
          timeoutSeconds: 5        # Timeout after 5s
          failureThreshold: 3      # Restart after 3 failures
        
        # Readiness probe (remove from load balancer if unhealthy)
        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 2
        
        # Startup probe (delay liveness/readiness for slow apps)
        startupProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 0
          periodSeconds: 5
          failureThreshold: 30  # Allow 150s for startup (30 * 5s)
        
        # Environment from ConfigMap/Secret
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: db-credentials
        
        # Graceful shutdown
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - sleep 15  # Allow in-flight requests to complete
      
      # Security context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      
      # Termination grace period
      terminationGracePeriodSeconds: 30
```

### Namespace Organization

Create `day5-namespace.yaml`:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    environment: staging
```

Deploy to specific namespace:
```bash
kubectl apply -f day5-namespace.yaml

# Deploy to production namespace
kubectl apply -f day5-production-deployment.yaml -n production

# Set default namespace
kubectl config set-context --current --namespace=production

# Verify
kubectl get pods  # Now shows production namespace by default
```

### Resource Quotas (Prevent Resource Exhaustion)

Create `day5-quota.yaml`:
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"       # Max 10 CPU cores requested
    requests.memory: 20Gi    # Max 20GB memory requested
    limits.cpu: "20"         # Max 20 CPU cores limit
    limits.memory: 40Gi      # Max 40GB memory limit
    persistentvolumeclaims: "10"  # Max 10 PVCs
    services.loadbalancers: "3"   # Max 3 LoadBalancers
```

### Horizontal Pod Autoscaler

Create `day5-hpa.yaml`:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapi-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapi
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Scale up if >70% CPU
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80  # Scale up if >80% memory
```

Enable metrics server (required for HPA):
```bash
# Minikube
minikube addons enable metrics-server

# Docker Desktop
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify
kubectl top nodes
kubectl top pods
```

### Organized Directory Structure

```
/kubernetes
  /base
    deployment.yaml
    service.yaml
    configmap.yaml
  /overlays
    /dev
      kustomization.yaml
      dev-config.yaml
    /staging
      kustomization.yaml
      staging-config.yaml
    /production
      kustomization.yaml
      production-config.yaml
      hpa.yaml
      quota.yaml
```

Example `kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
- ../../base
- hpa.yaml
- quota.yaml

replicas:
- name: myapi
  count: 5

images:
- name: myapp
  newTag: v1.2.0
```

Deploy with Kustomize:
```bash
kubectl apply -k kubernetes/overlays/production
```

### Validation Checklist
- [ ] Health checks responding (`kubectl describe pod`)
- [ ] Resource limits enforced
- [ ] HPA scaling based on load (`kubectl get hpa`)
- [ ] Namespace isolation working
- [ ] Graceful shutdown on pod termination

### Common Issues

**Issue**: Liveness probe failing, pod restarting constantly  
**Solution**: Increase `initialDelaySeconds` or `failureThreshold`

**Issue**: HPA not scaling  
**Solution**: Check metrics-server is running
```bash
kubectl top pods
# If error, metrics-server not installed
```

**Issue**: Cannot exceed resource quota  
**Solution**: Check quota usage
```bash
kubectl describe quota production-quota -n production
```

### Key Takeaways
- **Liveness probe**: Restart unhealthy pods
- **Readiness probe**: Remove from Service until ready
- **Resource limits**: Prevent one app consuming all resources
- **HPA**: Auto-scale based on CPU/memory
- **Namespaces**: Isolate environments (dev/staging/prod)

---

## Essential Commands Cheat Sheet

### Cluster Management
```bash
# Check cluster status
kubectl cluster-info
kubectl get nodes

# Switch context (cluster)
kubectl config get-contexts
kubectl config use-context docker-desktop
```

### Resource Management
```bash
# Create/Update
kubectl apply -f file.yaml
kubectl apply -f directory/
kubectl apply -k overlays/production  # Kustomize

# Delete
kubectl delete -f file.yaml
kubectl delete deployment myapi
kubectl delete pod myapi-abc123
kubectl delete all --all  # Delete everything in namespace

# Get resources
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get all  # All resources

# Watch changes
kubectl get pods --watch
```

### Inspection
```bash
# Describe (detailed info + events)
kubectl describe pod myapi-abc123
kubectl describe deployment myapi
kubectl describe node minikube

# Logs
kubectl logs myapi-abc123
kubectl logs -f myapi-abc123  # Follow (stream)
kubectl logs myapi-abc123 --previous  # Previous crashed container
kubectl logs -l app=myapi  # All pods with label

# Execute commands in pod
kubectl exec myapi-abc123 -- ls /app
kubectl exec -it myapi-abc123 -- /bin/bash
```

### Debugging
```bash
# Port forward (local testing)
kubectl port-forward pod/myapi-abc123 8080:80
kubectl port-forward service/myapi-service 8080:80

# Copy files
kubectl cp myapi-abc123:/app/logs/app.log ./app.log
kubectl cp ./config.json myapi-abc123:/app/config.json

# Events
kubectl get events --sort-by=.metadata.creationTimestamp

# Resource usage
kubectl top nodes
kubectl top pods
```

### Scaling & Updates
```bash
# Scale
kubectl scale deployment myapi --replicas=5

# Rolling update
kubectl set image deployment/myapi api=myapp:v2
kubectl rollout status deployment/myapi
kubectl rollout history deployment/myapi
kubectl rollout undo deployment/myapi

# Restart
kubectl rollout restart deployment/myapi
```

### Configuration
```bash
# ConfigMap
kubectl create configmap app-config --from-file=config.json
kubectl get configmap app-config -o yaml

# Secret
kubectl create secret generic db-creds --from-literal=password=secret
kubectl get secret db-creds -o yaml
```

### Namespace Management
```bash
# List namespaces
kubectl get namespaces

# Create namespace
kubectl create namespace dev

# Set default namespace
kubectl config set-context --current --namespace=dev

# Resource in specific namespace
kubectl get pods -n production
kubectl get pods --all-namespaces
```

### Quick Debugging Commands
```bash
# Why isn't my pod starting?
kubectl describe pod <pod-name>
# Look at Events section

# Why can't I access my service?
kubectl get endpoints myapi-service
# Should match pod IPs

# Is my config correct?
kubectl get configmap app-config -o yaml
kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 --decode

# What's consuming resources?
kubectl top nodes
kubectl top pods --all-namespaces --sort-by=memory
```

---

## Troubleshooting Guide

### Pod Issues

#### Pod Stuck in `Pending`
**Symptoms**: `kubectl get pods` shows `Pending` status

**Diagnosis**:
```bash
kubectl describe pod <pod-name>
```

**Common Causes**:
1. **Insufficient resources**
   ```
   Events: 0/1 nodes are available: 1 Insufficient cpu
   ```
   **Solution**: Reduce resource requests or add nodes

2. **PVC not bound**
   ```
   Events: pod has unbound immediate PersistentVolumeClaims
   ```
   **Solution**: Check PVC status
   ```bash
   kubectl get pvc
   kubectl describe pvc <pvc-name>
   ```

3. **ImagePullBackOff**
   ```
   Events: Failed to pull image "myapp:latest": image not found
   ```
   **Solution**: Verify image exists, check image name/tag

#### Pod CrashLoopBackOff
**Symptoms**: Pod restarts repeatedly

**Diagnosis**:
```bash
kubectl logs <pod-name>
kubectl logs <pod-name> --previous  # Logs from crashed container
kubectl describe pod <pod-name>
```

**Common Causes**:
1. **Application error**: Check logs for exceptions
2. **Liveness probe failing**: Increase `initialDelaySeconds`
3. **Missing configuration**: Verify ConfigMap/Secret exists

#### Pod Running but Not Ready
**Symptoms**: `READY` shows `0/1`

**Diagnosis**:
```bash
kubectl describe pod <pod-name>
# Check Readiness probe section
```

**Solution**: Readiness probe failing
- Check `/ready` endpoint exists
- Increase `initialDelaySeconds`
- Verify application starts correctly

### Service Issues

#### Cannot Access Service Externally
**Symptoms**: `curl http://localhost` times out

**Diagnosis**:
```bash
kubectl get service myapi-service
# Check EXTERNAL-IP column
```

**Solutions**:
1. **LoadBalancer pending (Minikube)**:
   ```bash
   minikube service myapi-service --url
   ```

2. **LoadBalancer pending (Kind)**:
   ```bash
   kubectl port-forward service/myapi-service 8080:80
   curl http://localhost:8080
   ```

3. **Wrong Service type**:
   Change `type: ClusterIP` to `type: LoadBalancer`

#### Service Exists but No Traffic Reaches Pods
**Diagnosis**:
```bash
kubectl get endpoints myapi-service
# Should show pod IPs
```

**Solution**: Selector mismatch
```bash
# Check Service selector
kubectl get service myapi-service -o yaml | grep -A5 selector

# Check Pod labels
kubectl get pods --show-labels

# Selectors must match pod labels exactly
```

### Configuration Issues

#### ConfigMap/Secret Changes Not Reflected
**Symptoms**: Updated ConfigMap but pods still use old values

**Solution**: Restart pods
```bash
kubectl rollout restart deployment myapi
```

**Better Solution**: Use ConfigMap hash in annotations (auto-restart on change)
```yaml
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

#### Secret Not Found Error
**Diagnosis**:
```bash
kubectl get secrets
kubectl describe pod <pod-name>
```

**Solution**: Create secret first
```bash
kubectl create secret generic db-creds --from-literal=password=secret
```

### Performance Issues

#### Pods Using Too Much Memory
**Diagnosis**:
```bash
kubectl top pods
kubectl describe pod <pod-name> | grep -A5 Limits
```

**Solution**: Set memory limits
```yaml
resources:
  limits:
    memory: "512Mi"
```

#### Application Slow/Unresponsive
**Diagnosis**:
```bash
kubectl top pods
kubectl top nodes
kubectl describe node <node-name>
```

**Solution**: Increase CPU requests or scale replicas
```bash
kubectl scale deployment myapi --replicas=5
```

### Storage Issues

#### PVC Stuck in Pending
**Diagnosis**:
```bash
kubectl describe pvc <pvc-name>
```

**Common Causes**:
1. **No StorageClass available**:
   ```bash
   kubectl get storageclass
   ```
   **Solution**: Install default storage class (Minikube: `minikube addons enable default-storageclass`)

2. **Insufficient storage**:
   **Solution**: Reduce storage request or add storage

#### Data Not Persisting
**Diagnosis**:
```bash
kubectl get pvc
kubectl get pv
```

**Solution**: Verify PVC is bound and mounted correctly
```bash
kubectl describe pod <pod-name> | grep -A10 Mounts
```

### Common Error Messages

| Error | Meaning | Solution |
|---|---|---|
| `ImagePullBackOff` | Cannot pull container image | Check image name/tag, verify registry access |
| `CrashLoopBackOff` | Container keeps crashing | Check logs: `kubectl logs <pod>` |
| `CreateContainerConfigError` | Invalid container config | Check ConfigMap/Secret references |
| `Insufficient cpu/memory` | Not enough resources | Reduce requests or add nodes |
| `Pending` | Waiting for scheduling | Check resources, PVC, node affinity |
| `ErrImagePull` | Image pull failed | Verify image exists and is accessible |
| `OOMKilled` | Out of memory | Increase memory limits |

### Getting Help

```bash
# Explain resource
kubectl explain pod
kubectl explain pod.spec.containers

# API reference
kubectl api-resources
kubectl api-versions

# Verbose output
kubectl get pods -v=8  # Debug level
```

---

## Next Steps

### Week 2: Advanced Topics
1. **Ingress Controllers**: HTTP(S) routing (replace multiple LoadBalancers)
2. **Helm**: Package manager (templated YAML, reusable charts)
3. **Init Containers**: Pre-start tasks (wait for dependencies, run migrations)
4. **DaemonSets**: Run on every node (logging agents, monitoring)
5. **Network Policies**: Firewall rules between pods

### Week 3: Production Readiness
1. **Monitoring**: Prometheus + Grafana
2. **Logging**: EFK stack (Elasticsearch, Fluentd, Kibana)
3. **CI/CD Integration**: GitHub Actions, Azure DevOps
4. **Service Mesh**: Istio/Linkerd for advanced networking
5. **Secrets Management**: External Secrets Operator, Azure Key Vault

### Cloud Migration
- **Azure Kubernetes Service (AKS)**: Managed Kubernetes on Azure
- **Amazon EKS**: Managed Kubernetes on AWS
- **Google GKE**: Managed Kubernetes on Google Cloud

Local cluster → Cloud cluster is straightforward:
```bash
# Update image registry
image: myregistry.azurecr.io/myapp:v1

# Deploy same YAML to cloud cluster
kubectl apply -f deployment.yaml
```

### Recommended Resources
- **Official Docs**: https://kubernetes.io/docs/
- **Interactive Tutorial**: https://kubernetes.io/docs/tutorials/kubernetes-basics/
- **kubectl Cheat Sheet**: https://kubernetes.io/docs/reference/kubectl/cheatsheet/
- **Kubernetes Patterns**: https://k8spatterns.io/

---

## Summary

### Key Concepts Learned
1. **Deployment**: Manages stateless applications (auto-healing, scaling)
2. **Service**: Provides stable networking (load balancing, DNS)
3. **ConfigMap/Secret**: Externalizes configuration
4. **StatefulSet**: Manages stateful applications (databases)
5. **PersistentVolume**: Durable storage
6. **Job/CronJob**: One-time and scheduled tasks
7. **Namespace**: Environment isolation

### Docker → Kubernetes Migration Checklist
- [ ] Convert `docker-compose.yml` to Deployment + Service YAMLs
- [ ] Move environment variables to ConfigMap/Secret
- [ ] Add resource limits (requests/limits)
- [ ] Implement health checks (liveness/readiness probes)
- [ ] Configure persistent storage for databases
- [ ] Set up namespaces for environments (dev/staging/prod)
- [ ] Add horizontal pod autoscaling
- [ ] Configure ingress for HTTP(S) routing (optional)

### Production Readiness Checklist
- [ ] Resource limits set on all containers
- [ ] Liveness/readiness probes configured
- [ ] Secrets stored securely (not hardcoded)
- [ ] Persistent storage for stateful apps
- [ ] Namespace isolation by environment
- [ ] Resource quotas to prevent exhaustion
- [ ] Horizontal pod autoscaling configured
- [ ] Graceful shutdown handling
- [ ] Logging aggregation set up
- [ ] Monitoring and alerting configured

You now have the foundation to deploy production-grade containerized applications on Kubernetes!

---

## Docker → Kubernetes Concept Map

### Core Concepts Translation

| **Docker Concept** | **Kubernetes Equivalent** | **Key Difference** |
|---|---|---|
| `docker run` | `kubectl apply -f deployment.yaml` | Declarative (desired state) vs imperative (one-time command) |
| Container | **Pod** | Pod = 1+ containers sharing network/storage |
| `docker-compose.yml` | **Deployment YAML** | Kubernetes maintains state automatically |
| `docker-compose up --scale api=3` | `replicas: 3` in Deployment | Built-in load balancing + health checks |
| Port mapping `-p 8080:80` | **Service** (type: LoadBalancer) | Abstracted networking with DNS |
| Volume `-v /data:/var/lib/data` | **PersistentVolumeClaim** | Storage abstracted from infrastructure |
| `docker network` | **Service** (ClusterIP) | Automatic DNS: `my-service.default.svc.cluster.local` |
| `restart: always` | **Deployment** with restartPolicy | Self-healing, auto-restarts crashed pods |
| `docker exec -it` | `kubectl exec -it pod-name -- bash` | Same concept, different CLI |
| `docker logs -f` | `kubectl logs -f pod-name` | Centralized logging |

### Example: docker-compose.yml → Kubernetes

**Docker Compose**:
```yaml
version: '3'
services:
  api:
    image: myapp:latest
    ports:
      - "8080:80"
    environment:
      - DB_HOST=db
      - DB_PASSWORD=secret
    restart: always
    deploy:
      replicas: 3
  
  db:
    image: postgres:15
    volumes:
      - dbdata:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=secret

volumes:
  dbdata:
```

**Kubernetes Equivalent** (we'll build this step-by-step):
- `api` service → **Deployment** + **Service**
- `db` service → **StatefulSet** + **Service** + **PersistentVolumeClaim**
- `environment` → **ConfigMap** + **Secret**
- `deploy.replicas` → `spec.replicas`