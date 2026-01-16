# Kubernetes fundamentals

## Table of Contents
- [Prerequisites & Setup](#prerequisites--setup)
- [Practice](#practice)
- [Exercise 1 — First Deployment](#exercise-1-first-deployment)
- [Exercise 2 — Configuration Management](#exercise-2-configuration-management)
- [Exercise 3 — Persistent Storage](#exercise-3-persistent-storage)
- [Exercise 4 — Scheduled Jobs](#exercise-4-scheduled-jobs)
- [Exercise 5 — Production Patterns](#exercise-5-production-patterns)
- [Essential Commands Cheat Sheet](#essential-commands-cheat-sheet)
- [Troubleshooting Guide](#troubleshooting-guide)
- [Next Steps & Summary](#summary)

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
Transform basic deployments into production-grade services by adding 5 critical patterns.

#### Step 5.1: Add Health Checks and Graceful Shutdown

##### Goal
Prevent traffic to unhealthy pods and allow in-flight requests to complete during restarts.
Four concepts: 

- Health checks:
    - Startup: Has the application started successfully?
    - Liveness: Is the application alive?
    - Readiness: Is the application ready to serve traffic?

- Graceful shutdown: Clean termination with zero dropped requests.

###### Step 5.1.1: Startup probe

Runs first and delays liveness/readiness checks.
Fails if app doesn't start within failureThreshold * periodSeconds (6 * 5s = 30s)
(here, we simulate a slow startup).

Create `exercise5-startup-probe.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapi-slow-start
  namespace: kubernetes-fundamentals
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapi-slow-start
  template:
    metadata:
      labels:
        app: myapi-slow-start
    spec:
      containers:
      - name: api
        image: mcr.microsoft.com/dotnet/samples:aspnetapp
        ports:
        - name: http
          containerPort: 8080
        
        # Add 20s delay before app starts responding
        lifecycle:
          postStart:
            exec:
              command:
              - /bin/sh
              - -c
              - sleep 20
        
        startupProbe:
          httpGet:
            path: /
            port: http
          initialDelaySeconds: 0
          periodSeconds: 5
          failureThreshold: 6  # 30s total allowance
        
        livenessProbe:
          httpGet:
            path: /
            port: http
          initialDelaySeconds: 10
          periodSeconds: 10
```

```bash
# Watch pod startup process
kubectl get pods -n kubernetes-fundamentals -w

# Expected behavior:
# 1. Pod shows "Running" but NOT "Ready" (0/1)
# 2. After ~20s, startup probe succeeds
# 3. Pod becomes "Ready" (1/1)
# 4. Liveness/readiness probes take over

# Check probe events
kubectl describe pod myapi-slow-start-<pod-id> -n kubernetes-fundamentals | grep -A20 Events:

# Expected events should all be normal/successful
```

##### Step 5.1.2: Failed startup probe

```bash
# Reduce failure threshold to trigger startup failure
kubectl patch deployment myapi-slow-start -n kubernetes-fundamentals -p '{"spec":{"template":{"spec":{"containers":[{"name":"api","startupProbe":{"failureThreshold":2}}]}}}}'

# Watch pod fail to start
kubectl get pods -n kubernetes-fundamentals -w

# Expected: Pod restarts repeatedly (CrashLoopBackOff)
# After some attempts, the pod is terminated

# Cleanup
kubectl delete deployment myapi-slow-start -n kubernetes-fundamentals
```

###### Step 5.1.2: Liveness probe

Restarts pod if it becomes unhealthy (deadlock, infinite loop, corrupted state)
(if `failureThreshold` fails consecutive times).

Create `exercise5-liveness-probe.yaml`:
```yaml
# Deploy test pod with controllable health endpoint
apiVersion: v1
kind: Pod
metadata:
  name: liveness-test
  namespace: kubernetes-fundamentals
spec:
  containers:
  - name: api
    image: nginx:alpine
    ports:
    - containerPort: 80
    
    livenessProbe:
      httpGet:
        path: /health
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 10
      failureThreshold: 3  # Restart after 30s (3 * 10s)
    
    # Create health endpoint
    lifecycle:
      postStart:
        exec:
          command:
          - /bin/sh
          - -c
          - echo "OK" > /usr/share/nginx/html/health
``

```bash
# Verify pod is healthy
kubectl get pod liveness-test -n kubernetes-fundamentals

# NAME            READY   STATUS    RESTARTS   AGE
# liveness-test   1/1     Running   0          11s

# Simulate application deadlock by deleting health endpoint
kubectl exec liveness-test -n kubernetes-fundamentals -- rm /usr/share/nginx/html/health

# Watch liveness probe fail and pod restart
kubectl logs -f liveness-test -n kubernetes-fundamentals

# Expected sequence:
# 10.1.0.1 - - [16/Jan/2026:08:58:21 +0000] "GET /health HTTP/1.1" 200 3 "-" "kube-probe/1.34" "-"
# 10.1.0.1 - - [16/Jan/2026:08:58:31 +0000] "GET /health HTTP/1.1" 404 153 "-" "kube-probe/1.34" "-"
#2026/01/16 08:58:31 [error] 35#35: *9 open() "/usr/share/nginx/html/health" failed (2: No such file or directory), client: 10.1.0.1, server: localhost, request: "GET /health HTTP/1.1", host: "10.1.0.205:80"
#10.1.0.1 - - [16/Jan/2026:08:58:41 +0000] "GET /health HTTP/1.1" 404 153 "-" "kube-probe/1.34" "-"
#2026/01/16 08:58:41 [error] 35#35: *10 open() "/usr/share/nginx/html/health" failed (2: No such file or directory), client: 10.1.0.1, server: localhost, request: "GET /health HTTP/1.1", host: "10.1.0.205:80"
#10.1.0.1 - - [16/Jan/2026:08:58:51 +0000] "GET /health HTTP/1.1" 404 153 "-" "kube-probe/1.34" "-"
#2026/01/16 08:58:51 [error] 35#35: *11 open() "/usr/share/nginx/html/health" failed (2: No such file or directory), client: 10.1.0.1, server: localhost, request: "GET /health HTTP/1.1", host: "10.1.0.205:80"
#2026/01/16 08:58:51 [notice] 1#1: signal 3 (SIGQUIT) received, shutting down
#2026/01/16 08:58:51 [notice] 35#35: gracefully shutting down

# Verify restart count increased
kubectl get pod liveness-test -n kubernetes-fundamentals
# NAME            READY   STATUS    RESTARTS   AGE
# liveness-test   1/1     Running   1          2m

# Cleanup
kubectl delete pod liveness-test -n kubernetes-fundamentals
```

###### Step 5.1.3: Readiness probe

Readiness probe removes pod from Service endpoints if unhealthy.
**NOT restarted like liveness probe.**

Create `exercise5-readiness-probe.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: readiness-test
  namespace: kubernetes-fundamentals
spec:
  replicas: 2
  selector:
    matchLabels:
      app: readiness-test
  template:
    metadata:
      labels:
        app: readiness-test
    spec:
      containers:
      - name: api
        image: nginx:alpine
        ports:
        - containerPort: 80
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 2  # Remove from Service after 10s (2 * 5s)
        
        lifecycle:
          postStart:
            exec:
              command:
              - /bin/sh
              - -c
              - echo "READY" > /usr/share/nginx/html/ready
---
apiVersion: v1
kind: Service
metadata:
  name: readiness-test-service
  namespace: kubernetes-fundamentals
spec:
  selector:
    app: readiness-test
  ports:
  - port: 80
    targetPort: 80
```

```bash
# Check Service endpoints (should show 2 pod IPs)
kubectl get endpoints readiness-test-service -n kubernetes-fundamentals
# NAME                     ENDPOINTS                     AGE
# readiness-test-service   10.1.0.206:80,10.1.0.207:80   34s

# Get one pod name
POD_NAME=$(kubectl get pods -n kubernetes-fundamentals -l app=readiness-test -o jsonpath='{.items[0].metadata.name}')

# Simulate pod becoming unavailable (delete ready endpoint)
kubectl exec $POD_NAME -n kubernetes-fundamentals -- rm /usr/share/nginx/html/ready

# Watch pod status (stays Running, but becomes NOT Ready)
kubectl get pods -n kubernetes-fundamentals -l app=readiness-test -w

# Expected:
# readiness-test-xxx   0/1   Running   0   2m
# readiness-test-xxx   1/1   Running   0   2m

# Check Service endpoints (should show only 1 pod now)
kubectl get endpoints readiness-test-service -n kubernetes-fundamentals
# NAME                     ENDPOINTS
# readiness-test-service   10.1.0.6:80  (only healthy pod)

# Verify traffic only goes to healthy pod
kubectl run -it --rm curl-test --image=curlimages/curl --restart=Never -n kubernetes-fundamentals -- sh -c "for i in \$(seq 1 10); do curl -s http://readiness-test-service && echo; done"

# All 10 requests should succeed (routed only to healthy pod)

# Restore pod to healthy state
kubectl exec $POD_NAME -n kubernetes-fundamentals -- sh -c 'echo "READY" > /usr/share/nginx/html/ready'

# Watch pod become Ready again
kubectl get pods -n kubernetes-fundamentals -l app=readiness-test -w

# Check endpoints (should show 2 pods again)
kubectl get endpoints readiness-test-service -n kubernetes-fundamentals

# Check probe logs
kubectl logs -f <pod-id> -n kubernetes-fundamentals

# Cleanup
kubectl delete -f exercise5-readiness-probe.yaml
```

###### Step 5.1.4: Graceful shutdown

Allow in-flight requests to complete before terminating pod.

Create `exercise5-graceful-shutdown.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: graceful-simple
  namespace: kubernetes-fundamentals
spec:
  containers:
  - name: app
    image: busybox
    command:
    - /bin/sh
    - -c
    - |
      echo "Application started"
      trap "echo 'Received SIGTERM, shutting down gracefully...'; sleep 5; echo 'Shutdown complete'" TERM
      while true; do sleep 1; done
    
    lifecycle:
      preStop:
        exec:
          command:
          - /bin/sh
          - -c
          - |
            echo "PreStop hook triggered - waiting 10s for connection draining..."; sleep 10; echo "PreStop complete"
  
  terminationGracePeriodSeconds: 20
```

```bash
# View logs in real-time
kubectl logs graceful-simple -n kubernetes-fundamentals

# Delete pod and observe shutdown sequence
kubectl delete pod graceful-simple -n kubernetes-fundamentals

# Expected log output:
# Application started
# Received SIGTERM, shutting down gracefully...
# Shutdown complete
```

## Next Steps for Future Practice

### Advanced Topics
1. **Ingress Controllers**: HTTP(S) routing (replace multiple LoadBalancers)
2. **Helm**: Package manager (templated YAML, reusable charts)
3. **Init Containers**: Pre-start tasks (wait for dependencies, run migrations)
4. **DaemonSets**: Run on every node (logging agents, monitoring)
5. **Network Policies**: Firewall rules between pods

### Production Readiness
1. **Monitoring**: Prometheus + Grafana
2. **Logging**: EFK stack (Elasticsearch, Fluentd, Kibana)
3. **CI/CD Integration**: GitHub Actions, Azure DevOps
4. **Service Mesh**: Istio/Linkerd for advanced networking
5. **Secrets Management**: External Secrets Operator, Azure Key Vault

### Cloud Migration
- **Azure Kubernetes Service (AKS)**: Managed Kubernetes on Azure
- **Amazon EKS**: Managed Kubernetes on AWS
- **Google GKE**: Managed Kubernetes on Google Cloud

### Kubernetes patterns

---

## Summary

### Key Concepts Learned
1. **Deployment**: Manages stateless applications (auto-healing, scaling)
2. **Service**: Provides stable networking (load balancing, DNS)
3. **ConfigMap/Secret**: Externalizes configuration
4. **StatefulSet**: Manages stateful applications (databases)
5. **PersistentVolume**: Durable storage
6. **Job/CronJob**: One-time and scheduled tasks
7. **Probes**: Health checks for reliability

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