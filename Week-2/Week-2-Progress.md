
# Daily Progress Report — November 19, 2026

## Project: Highly Available Three-Tier Application on Kubernetes

**Week:** Week 2 — CI/CD and Database Migration  
**Date:** November 19, 2026  
**Status:** CloudNativePG setup completed; database migration in progress.

---

## 1. Installed CloudNativePG Operator

Installed CloudNativePG Operator version 1.30.2 using:

```bash
kubectl apply --server-side -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.30/releases/cnpg-1.30.2.yaml
```

### What was accomplished?

- Installed the CloudNativePG operator.
- Created the required Kubernetes Custom Resource Definitions (CRDs).
- Installed the controller and supporting RBAC resources.
- Verified that the controller Pod was running.

Verification:

```bash
kubectl get pods -n cnpg-system
```

Result: `cnpg-controller-manager` was running successfully (`1/1 Running`).

---

## 2. Created CloudNativePG PostgreSQL Cluster

Created the configuration file:

`k8s/cnpg-cluster.yaml`

Configured the following:

| Configuration | Value |
|---|---|
| Cluster name | `todo-postgres` |
| Namespace | `junior-devops` |
| PostgreSQL instances | 1 |
| Storage size | 1Gi |
| StorageClass | standard |
| CPU request and limit | 500m |
| Memory request and limit | 512Mi |

Validated and applied the configuration successfully.

### Cluster verification

```bash
kubectl get cluster -n junior-devops
```

The cluster reached:

- `Ready: True`
- `Ready Instances: 1`
- `Phase: Cluster in healthy state`

The primary PostgreSQL Pod:

`todo-postgres-1`

Status: `1/1 Running`

---

## 3. Verified Persistent Storage

Checked the PostgreSQL PersistentVolumeClaim:

```bash
kubectl get pvc -n junior-devops
```

Results:

| Property | Value |
|---|---|
| PVC | `todo-postgres-1` |
| Status | Bound |
| Capacity | 1Gi |
| Access Mode | RWO |
| StorageClass | standard |

This confirms that PostgreSQL has persistent storage allocated through Kubernetes.

---

## 4. Verified PostgreSQL Connectivity

Executed:

```bash
kubectl exec -it todo-postgres-1 -n junior-devops -- psql -U postgres -c "SELECT version();"
```

Successfully retrieved the PostgreSQL version:

**PostgreSQL 18.7**

This confirmed that the new PostgreSQL instance was operational and accepting local database queries.

---

## 5. Inspected CloudNativePG Services and Secrets

Executed:

```bash
kubectl get services,secrets -n junior-devops
```

Discovered the following CNPG services:

| Service | Purpose |
|---|---|
| `todo-postgres-rw` | Routes connections to the primary instance for read/write operations |
| `todo-postgres-ro` | Routes connections to read-only instances |
| `todo-postgres-r` | Routes connections to any available PostgreSQL instance |

Also discovered the generated application Secret:

`todo-postgres-app`

It contains the credentials for the default CNPG application user.

---

## 6. Inspected Existing PostgreSQL Database

Before migrating, inspected the existing Docker PostgreSQL database.

Database: `todo_db`  
Username: `todo_user`

Checked the number of records:

```bash
docker exec junior-devops-postgres psql -U todo_user -d todo_db -c "SELECT COUNT(*) FROM todos;"
```

Result: **4 records**

Listed the existing records:

```bash
docker exec junior-devops-postgres psql -U todo_user -d todo_db -c "SELECT id, task, completed, created_at FROM todos ORDER BY id;"
```

Existing Todo records:

| ID | Task | Completed |
|---|---|---|
| 2 | Test PostgreSQL persistence | False |
| 3 | learn Kubernetes | False |
| 4 | learn docker | False |
| 5 | CICD pipeline | False |

All four records were present in the original database.

---

## 7. Created Logical Database Backup

Created a backup directory and generated a PostgreSQL logical dump:

```bash
mkdir -p backups && docker exec junior-devops-postgres pg_dump -U todo_user -d todo_db --no-owner --no-privileges > backups/todo_db_backup.sql
```

Verified the backup:

```bash
ls -lh backups/todo_db_backup.sql
```

Result:

`backups/todo_db_backup.sql` — approximately 2.1 KB.

Inspected the backup contents:

```bash
grep -nE 'COPY public.todos|INSERT INTO|setval|restrict' backups/todo_db_backup.sql
```

Confirmed that the backup contains:

- Existing `todos` table data.
- PostgreSQL COPY data section.
- Sequence information.
- Sequence's last value, 5, meaning the next generated Todo ID should be 6.

**Important:** The original PostgreSQL Docker container and its persistent volume have not been deleted or modified as part of the migration.

The backup file should remain private and should not be committed to the public Git repository.

---

## 8. Inspected Existing Backend Database Configuration

Inspected:

`k8s/backend-deployment.yaml`

Confirmed that the backend currently expects:

| Environment Variable | Current Configuration |
|---|---|
| `DB_HOST` | `junior-devops-postgres` |
| `DB_PORT` | `5432` |
| `DB_NAME` | `todo_db` |
| `DB_USER` | `todo_user` |
| `DB_PASSWORD` | Retrieved from `junior-devops-db-secret` |

The application uses environment variables for database connectivity, so changing the database destination does not require rewriting the Python backend code.

---

## 9. Started Creating CNPG DatabaseRole

Created:

`k8s/cnpg-role.yaml`

The objective is to create a PostgreSQL user named:

`todo_user`

Initially, the configuration used `spec.password`, but Kubernetes rejected it:

```text
strict decoding error: unknown field "spec.password"
```

To understand the installed CRD, inspected its schema:

```bash
kubectl explain databaserole.spec --api-version=postgresql.cnpg.io/v1
```

Discovered that the correct supported field is:

`passwordSecret`

Further inspected it:

```bash
kubectl explain databaserole.spec.passwordSecret --api-version=postgresql.cnpg.io/v1
```

Confirmed that it requires the name of an existing Kubernetes Secret of type `kubernetes.io/basic-auth`, containing:

- `username`
- `password`

### Created the password Secret

Generated a random password without displaying it:

```bash
PASSWORD="$(openssl rand -base64 32)" && kubectl create secret generic todo-user-password --namespace junior-devops --type=kubernetes.io/basic-auth --from-literal=username=todo_user --from-literal=password="$PASSWORD" && unset PASSWORD
```

Result:

```text
secret/todo-user-password created
```

Updated `k8s/cnpg-role.yaml` to use:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: DatabaseRole

metadata:
  name: todo-user
  namespace: junior-devops

spec:
  name: todo_user

  cluster:
    name: todo-postgres

  login: true

  passwordSecret:
    name: todo-user-password
```

Successfully validated the corrected configuration using:

```bash
kubectl apply --dry-run=server -f k8s/cnpg-role.yaml
```

Result: **Dry-run successful.**

The actual `DatabaseRole` has not yet been applied.

---

## 10. Current Project Status

### Completed today

- [x] Installed CloudNativePG Operator 1.30.2.
- [x] Created the `todo-postgres` PostgreSQL cluster.
- [x] Verified PostgreSQL Pod health.
- [x] Verified persistent storage.
- [x] Tested PostgreSQL connectivity.
- [x] Inspected CNPG services and generated Secrets.
- [x] Inspected the original database.
- [x] Confirmed four existing Todo records.
- [x] Created a logical database backup.
- [x] Inspected backend database configuration.
- [x] Created the `todo-user-password` Kubernetes Secret.
- [x] Corrected and validated the DatabaseRole YAML.

### Pending

- [ ] Apply the `DatabaseRole` configuration.
- [ ] Verify the PostgreSQL user was created successfully.
- [ ] Create the `todo_db` database.
- [ ] Restore the existing four Todo records.
- [ ] Verify the restored data and sequence.
- [ ] Connect the backend to the CNPG PostgreSQL service.
- [ ] Test Todo CRUD operations through the application.
- [ ] Verify application functionality after database migration.

---

## 11. Next Session — Starting Point

Resume from **Step 29.7: Apply the DatabaseRole**.

Command:

```bash
kubectl apply -f k8s/cnpg-role.yaml
```

After applying, verify that CNPG reconciles the role successfully.

Then proceed with creating `todo_db`, restoring the backup, and connecting the backend to:

`todo-postgres-rw`

---

## 12. Important Notes

- Keep the original Docker PostgreSQL container and volume intact until migration is completely verified.
- Never expose Kubernetes Secret values or database passwords.
- Do not commit the database backup to a public repository.
- Preserve existing Minikube resources and the `devops-fresh` profile.
- Keep resource consumption low because the development laptop has 8 GB RAM.
- PostgreSQL currently has one instance. High availability through additional replicas remains pending.

**Today's milestone:** Successfully deployed and verified a Kubernetes-managed PostgreSQL cluster and prepared the existing Todo database for migration.

# Week 2 – Day 9: CloudNativePG Database Migration

**Date:** November 20, 2026  
**Project:** Highly Available Three-Tier Application on Kubernetes

## 1. Today's Objective

Migrate the existing Todo application's PostgreSQL database from a standalone Docker container to a Kubernetes-managed CloudNativePG cluster and connect the backend to the new database.

## 2. Create and Verify the Database

Created the CNPG Database resource using:

`k8s/cnpg-database.yaml`

Configuration:
- Database resource: `todo-db`
- PostgreSQL database: `todo_db`
- Owner: `todo_user`
- Cluster: `todo-postgres`
- Reclaim policy: `retain`

Commands:

```bash
kubectl apply --dry-run=server -f k8s/cnpg-database.yaml

kubectl apply -f k8s/cnpg-database.yaml

kubectl describe database todo-db -n junior-devops
```

**Result:** Database successfully created. `Applied: true`.

## 3. Verify the Empty Database

Before migration, checked whether the target database contained tables.

```bash
kubectl exec -it todo-postgres-1 -n junior-devops -- psql -U postgres -d todo_db -c "\dt"
```

Result: No tables found. Database was ready for restoration.

## 4. Restore the PostgreSQL Backup

Previously created backup:

`backups/todo_db_backup.sql`

Restored the backup into CloudNativePG using:

```bash
{ printf 'SET ROLE todo_user;\n'; cat backups/todo_db_backup.sql; } | kubectl exec -i todo-postgres-1 -n junior-devops -- psql -v ON_ERROR_STOP=1 -U postgres -d todo_db
```

### Important operations

- `SET ROLE todo_user`: Ensures restored database objects belong to the application role.
- `CREATE TABLE`: Created the Todo table.
- `CREATE SEQUENCE`: Created the ID sequence.
- `COPY 4`: Restored four Todo records.
- `setval = 6`: Restored the sequence position.

**Result:** Backup successfully restored without SQL errors.

## 5. Verify Migrated Records

Command:

```bash
kubectl exec -it todo-postgres-1 -n junior-devops -- psql -U postgres -d todo_db -c "SELECT id, task, completed FROM todos ORDER BY id;"
```

Verified records:

| ID | Task | Completed |
|---|---|---|
| 2 | Test PostgreSQL persistence | false |
| 3 | learn Kubernetes | false |
| 4 | learn docker | false |
| 5 | CICD pipeline | false |

**Result:** All four original records successfully migrated.

## 6. Connect Backend to CloudNativePG

Updated:

`k8s/backend-deployment.yaml`

Updated database environment variables:

```yaml
env:
  - name: DB_HOST
    value: todo-postgres-rw
  - name: DB_PORT
    value: "5432"
  - name: DB_NAME
    value: todo_db
  - name: DB_USER
    value: todo_user
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: todo-user-password
        key: password
```

The backend now connects to the CNPG read/write service instead of the old standalone PostgreSQL service.

Validated and applied:

```bash
kubectl apply --dry-run=server -f k8s/backend-deployment.yaml

kubectl apply -f k8s/backend-deployment.yaml
```

## 7. Verify Backend Deployment

Checked rollout:

```bash
kubectl rollout status deployment/junior-devops-backend -n junior-devops
```

Result: Deployment successfully rolled out.

Checked Pods:

```bash
kubectl get pods -n junior-devops
```

Observed status:

| Component | Ready | Restarts |
|---|---|---|
| Backend | 1/1 | 0 |
| Frontend | 1/1 | 11 |
| CNPG PostgreSQL | 1/1 | 7 |

Backend was running successfully.

## 8. Check Backend Logs

Command:

```bash
kubectl logs deployment/junior-devops-backend -n junior-devops --tail=50
```

Result:
- Flask application started successfully.
- Running on port 5001.
- No startup errors shown.

## 9. Test Backend and Database Connectivity

Executed the Todo API from inside the backend Pod:

```bash
kubectl exec deployment/junior-devops-backend -n junior-devops -- python -c "import urllib.request; r=urllib.request.urlopen('http://127.0.0.1:5001/todos'); print('HTTP Status:', r.status); print(r.read().decode())"
```

Output:

```json
{
  "todos": [
    {"completed": false, "id": 2, "task": "Test PostgreSQL persistence"},
    {"completed": false, "id": 3, "task": "learn Kubernetes"},
    {"completed": false, "id": 4, "task": "learn docker"},
    {"completed": false, "id": 5, "task": "CICD pipeline"}
  ]
}
```

**HTTP Status: 200**

Result:
- Backend successfully connected to CloudNativePG.
- All four migrated records were returned.
- Original record IDs were preserved.
- Database authentication and connectivity worked.

## 10. Pending Tasks

1. Investigate why the CNPG PostgreSQL Pod has restarted seven times.
2. Check PostgreSQL Pod events, previous container state, and exit reason.
3. Test creating new Todo records through the API.
4. Test updating and deleting Todo records.
5. Verify the frontend works with the migrated database.
6. Verify data persistence after backend and database restarts.
7. Keep the original Docker PostgreSQL container and volume untouched until the migration is completely verified.

## 11. Today's Final Outcome

**Successfully migrated the existing Todo database from standalone Docker PostgreSQL to CloudNativePG in Kubernetes.**

The Kubernetes backend now communicates with the new CNPG database and successfully retrieves all four migrated Todo records.

**Next session:** Investigate PostgreSQL Pod restarts and perform complete CRUD and persistence testing.

---

**Week 2 Progress: CNPG database migration and backend integration completed successfully.**


# DevOps Project Progress — 23 November 2026

## Today's Objective

Prepare the backend Deployment for Kubernetes Horizontal Pod Autoscaling (HPA) by configuring CPU and memory resources, enabling Metrics Server, and verifying Kubernetes resource metrics.

## 1. Backend Resource Configuration

Initially checked the backend Deployment's resource configuration:

```bash
kubectl get deployment junior-devops-backend -n junior-devops -o jsonpath='{.spec.template.spec.containers[*].resources}'
```

**Output:** `{}`

This confirmed that CPU and memory requests and limits were not configured.

Updated `k8s/backend-deployment.yaml` with the following settings:

| Resource | Value |
|---|---|
| Replicas | 2 |
| CPU Request | 100m |
| CPU Limit | 500m |
| Memory Request | 128Mi |
| Memory Limit | 256Mi |

Preserved the existing:
- Backend Docker image
- Container port (5001)
- PostgreSQL connection configuration
- Kubernetes Secret reference for the database password

Applied the updated Deployment:

```bash
kubectl apply -f k8s/backend-deployment.yaml
```

Checked the rollout:

```bash
kubectl rollout status deployment/junior-devops-backend -n junior-devops
```

**Result:** Backend Deployment successfully rolled out.

## 2. Enabled Metrics Server

Checked whether Metrics Server was already installed:

```bash
kubectl get deployment metrics-server -n kube-system
```

Metrics Server was not found.

Enabled the Minikube Metrics Server addon:

```bash
minikube addons enable metrics-server -p devops-fresh
```

**Result:** Metrics Server addon successfully enabled.

Initially, the Metrics API was unavailable. Checked the Metrics Server Pod:

```bash
kubectl get pods -n kube-system | grep metrics-server
```

Output:

```text
metrics-server-9d74bb658-vmrz6   1/1   Running   0   100s
```

The Metrics Server Pod was running successfully.

## 3. Verified Kubernetes Resource Metrics

Executed:

```bash
kubectl top pods -n junior-devops
```

Successfully retrieved CPU and memory usage.

Observed output:

| Pod | CPU Usage | Memory Usage |
|---|---:|---:|
| Backend Replica 1 | 1m | 31Mi |
| Backend Replica 2 | 1m | 31Mi |
| Frontend Replica 1 | 0m | 10Mi |
| Frontend Replica 2 | 0m | 12Mi |
| todo-postgres-1 | 24m | 88Mi |
| todo-postgres-2 | 24m | 108Mi |

**Result:** Metrics Server is working, and Kubernetes can retrieve live CPU and memory metrics from the Pods.

## 4. Current Project Status

- Backend Deployment configured with 2 replicas.
- CPU and memory requests configured.
- CPU and memory limits configured.
- Backend Deployment rollout completed successfully.
- Minikube Metrics Server addon enabled.
- Metrics Server Pod running successfully.
- Kubernetes Metrics API working.
- `kubectl top pods` successfully displaying resource usage.

## 5. Next Task — Horizontal Pod Autoscaling (HPA)

The next planned task is to configure Horizontal Pod Autoscaling for the backend.

Planned HPA configuration:

| Setting | Value |
|---|---|
| Target Deployment | junior-devops-backend |
| Minimum Replicas | 2 |
| Maximum Replicas | 4 |
| Target CPU Utilization | 50% |

HPA will automatically scale backend Pods based on CPU utilization.

**Note:** HPA configuration has not yet been created or applied. It is pending for the next session.

## Commands Completed Today

```bash
kubectl get deployment junior-devops-backend -n junior-devops -o jsonpath='{.spec.template.spec.containers[*].resources}'

kubectl apply -f k8s/backend-deployment.yaml

kubectl rollout status deployment/junior-devops-backend -n junior-devops

kubectl get deployment metrics-server -n kube-system

minikube addons enable metrics-server -p devops-fresh

kubectl get pods -n kube-system | grep metrics-server

kubectl top pods -n junior-devops
```

---

**Today's Achievement:** Successfully configured backend resource requests and limits, enabled Metrics Server, and verified live Kubernetes resource metrics. Ready to proceed with HPA configuration in the next session.


# Week 2 – Day 2 Progress Report

**Date:** November 24, 2026  
**Project:** Highly Available Three-Tier Application on Kubernetes  
**Status:** Database setup and controlled switchover completed

---

## 1. Today's Objectives

Today's work focused on completing the PostgreSQL High Availability setup using CloudNativePG (CNPG), configuring database replicas, and testing a controlled primary switchover.

## 2. Minikube Environment

Started the existing Minikube cluster:

```bash
minikube start -p devops-fresh
```

Enabled Metrics Server:

```bash
minikube addons enable metrics-server -p devops-fresh
```

Checked resource utilization:

```bash
kubectl top nodes
kubectl top pods -n junior-devops
```

Checked node capacity and resource allocation:

```bash
kubectl describe node devops-fresh
```

The node was Ready, with MemoryPressure and DiskPressure conditions set to False.

## 3. Horizontal Pod Autoscaling (HPA)

Configured HPA for the backend deployment.

**Configuration:**
- Minimum replicas: 2
- Maximum replicas: 4
- Target CPU utilization: 50%

**Testing performed:**
- Increased backend CPU utilization using a load test.
- Observed the backend scale from 2 to 4 replicas.
- Stopped the load test.
- Observed the replicas scale back from 4 to 2.

HPA scaling was successfully tested.

## 4. CloudNativePG – Three Database Instances

Updated the CNPG cluster configuration in:

`k8s/cnpg-cluster.yaml`

Changed the instance count to:

```yaml
spec:
  instances: 3
```

Applied the configuration:

```bash
kubectl apply -f k8s/cnpg-cluster.yaml
```

**Final database architecture:**

| Instance | Role |
|---|---|
| todo-postgres-1 | Replica |
| todo-postgres-2 | Primary |
| todo-postgres-3 | Replica |

The cluster contains:
- 1 Primary PostgreSQL instance.
- 2 Standby PostgreSQL instances.
- Streaming replication between the primary and replicas.

Verified replication:

```bash
kubectl exec -n junior-devops todo-postgres-1 -- psql -U postgres -d postgres -c "SELECT application_name, state, sync_state FROM pg_stat_replication;"
```

Both replicas were in `streaming` state with asynchronous replication.

## 5. CNPG Status Verification

Installed the CloudNativePG kubectl plugin (version 1.30.2).

Checked the cluster status:

```bash
kubectl cnpg status todo-postgres -n junior-devops
```

**Verified results:**
- Total instances: 3
- Ready instances: 3
- Cluster status: Healthy
- One Primary instance.
- Two Standby instances.
- Both replicas streaming.
- Zero replication lag at the time of verification.

## 6. Controlled Primary Switchover

Initially, the primary instance was:

`todo-postgres-1`

Performed a controlled switchover by promoting the second instance:

```bash
kubectl cnpg promote -n junior-devops todo-postgres todo-postgres-2
```

Checked the cluster status again:

```bash
kubectl cnpg status todo-postgres -n junior-devops
```

**Result after switchover:**

| Instance | Updated Role |
|---|---|
| todo-postgres-1 | Standby |
| todo-postgres-2 | Primary |
| todo-postgres-3 | Standby |

The switchover was successful.

The cluster returned to a healthy state with all three instances ready.

**Important:** This was a planned, controlled switchover. An unexpected primary failure and automatic failover have not yet been tested.

## 7. Backend Database Connectivity

The backend connects to PostgreSQL through the CNPG read-write service:

```text
DB_HOST=todo-postgres-rw
DB_PORT=5432
DB_NAME=todo_db
DB_USER=todo_user
```

The database password is provided through a Kubernetes Secret.

After the primary changed from `todo-postgres-1` to `todo-postgres-2`, the backend continued using the same service name.

Verified that the backend could still retrieve the existing Todo records after the switchover.

**Result:** Backend database connectivity and existing data access were successful.

## 8. Current Database Architecture

```text
             Frontend
                 |
                 v
          Flask Backend
          (2 Replicas)
                 |
                 v
        todo-postgres-rw
                 |
                 v
       CloudNativePG Cluster
                 |
       +---------+---------+
       |         |         |
       v         v         v
    Primary   Replica   Replica
    Instance  Instance  Instance
       2         1         3
```

All three PostgreSQL instances are currently running on the same Minikube node.

This demonstrates database replication and controlled primary switchover, but does not provide protection against failure of the entire Kubernetes node.

## 9. Today's Achievements

- [x] Started the existing Minikube environment.
- [x] Enabled Metrics Server.
- [x] Configured and tested backend HPA.
- [x] Increased CNPG cluster size to three instances.
- [x] Configured one primary and two standby replicas.
- [x] Verified PostgreSQL streaming replication.
- [x] Installed the CNPG kubectl plugin.
- [x] Verified the cluster health.
- [x] Successfully performed a controlled primary switchover.
- [x] Verified backend access to existing database records after switchover.

## 10. Remaining Week 2 Tasks

The following work is planned for the next session:

- [ ] Configure GitLab CI/CD pipeline.
- [ ] Automate Docker image building.
- [ ] Automate Docker image publishing.
- [ ] Verify the remaining Week 2 deliverables.
- [ ] Update project documentation.

---

**End of Day 2 – November 24, 2026**

**Progress:** PostgreSQL High Availability setup, replication, controlled switchover, and backend data verification completed.

# Week 2 – Day 3 Progress

**Date:** 25 November 2026  
**Project:** Highly Available Three-Tier Application on Kubernetes  
**Focus:** GitLab Runner Installation and Minikube Network Troubleshooting

## 1. GitLab Runner Installation

- Updated Ubuntu package information using `sudo apt-get update`.
- Added the official GitLab Runner package repository.
- Installed GitLab Runner version **19.6.1** successfully.
- GitLab Runner is installed but has not yet been registered with the GitLab project.

Installation command:

```bash
sudo apt-get install gitlab-runner
```

## 2. Minikube Startup Issue

After installing GitLab Runner, its installation cleanup process deleted several unused Docker networks, including the `devops-fresh` network.

When attempting to start the existing Minikube profile, the following error occurred:

```text
failed to set up container networking:
network ... not found
```

This happened because the existing Minikube container was configured to use the deleted Docker network.

## 3. Troubleshooting Performed

### Step 1: Inspected the existing Minikube container

```bash
docker inspect devops-fresh --format '{{json .NetworkSettings.Networks}}'
```

Confirmed that:
- The original Minikube container still exists.
- Its previous IP configuration is `192.168.58.2`.
- Its Docker network reference was missing.

### Step 2: Inspected container labels

```bash
docker inspect devops-fresh --format 'Labels: {{json .Config.Labels}}'
```

Confirmed that the container belongs to the original Minikube profile `devops-fresh`.

### Step 3: Checked available Docker networks

```bash
docker network ls
```

Observed that the `devops-fresh` network was missing.

### Step 4: Inspected Minikube configuration

```bash
cat ~/.minikube/profiles/devops-fresh/config.json
```

Verified the existing configuration:

- Profile: `devops-fresh`
- Driver: Docker
- Kubernetes version: `v1.35.1`
- CPU: 2
- Memory: 2048 MB
- Existing node IP: `192.168.58.2`
- Nodes: 1

### Step 5: Checked network routes

```bash
ip route
```

Verified that the WSL network uses `172.19.64.0/20`.

### Step 6: Inspected existing Docker network subnets

```bash
docker network inspect $(docker network ls -q) --format '{{.Name}}: {{range .IPAM.Config}}{{.Subnet}} gateway={{.Gateway}}{{end}}'
```

Existing networks included:

- bridge: `172.17.0.0/16`
- ecommerce-devops-platform_default: `172.24.0.0/16`
- junior-devops-network: `172.25.0.0/16`

The original Minikube subnet `192.168.58.0/24` was not in use by these networks.

### Step 7: Recreated the missing Docker network

Executed:

```bash
docker network create \
  --driver=bridge \
  --subnet=192.168.58.0/24 \
  --gateway=192.168.58.1 \
  devops-fresh
```

The Docker network was successfully created.

## 4. Current Status

### Completed Today

- GitLab Runner v19.6.1 installed.
- Official GitLab Runner repository configured.
- Identified the missing Minikube Docker network.
- Inspected the original Minikube container and configuration.
- Checked existing Docker subnet allocations.
- Recreated the `devops-fresh` Docker network.

### Pending

- Successfully start the existing Minikube cluster.
- Verify Kubernetes nodes and existing workloads.
- Register GitLab Runner with the GitLab project.
- Create the application Helm chart.
- Configure GitLab Container Registry.
- Create the GitLab CI/CD pipeline.

## 5. Next Session Plan

1. Continue troubleshooting Minikube startup using the latest error output.
2. Restore access to the existing Kubernetes cluster without deleting the profile.
3. Verify existing application and database resources.
4. Register GitLab Runner.
5. Prepare the Helm chart.
6. Configure the GitLab CI/CD pipeline.

## Important Notes

- Do not delete or recreate the existing Minikube profile.
- Preserve existing application containers, database data and Docker volumes.
- The original Minikube container and profile configuration were found to be intact during inspection.
- The Docker network was recreated, but successful Minikube startup has not yet been verified.
- GitLab Runner is installed but CI/CD execution has not started.

**Today's Outcome:** GitLab Runner installation was completed, and the missing Docker network was restored. Minikube startup troubleshooting remains in progress.


# Week 2 – Day 4: GitLab Runner & Kubernetes RBAC Configuration

**Date:** November 26, 2026  
**Project:** Highly Available Three-Tier Application on Kubernetes  
**Focus:** GitLab Runner integration with Kubernetes and RBAC security configuration

---

## 1. Today's Objective

The main objective was to configure GitLab Runner to communicate with our local Minikube Kubernetes cluster securely.

We want GitLab Runner to execute our CI/CD pipeline, build Docker images, and deploy our application to Kubernetes using Helm.

Before allowing automated deployments, we needed to configure the required tools and Kubernetes permissions.

## 2. Verified Required Tools

We verified that the GitLab Runner Linux user could access the following tools.

| Tool | Purpose | Status |
|---|---|---|
| Docker | Build container images | Verified |
| kubectl | Communicate with Kubernetes | Verified |
| Helm v4.3.0 | Deploy applications using Helm | Verified |

### Verify Docker access

```bash
sudo -u gitlab-runner docker version
```

### Verify kubectl access

```bash
sudo -u gitlab-runner kubectl --kubeconfig=/etc/gitlab-runner/.kube/config get nodes
```

### Verify Helm access

```bash
sudo -u gitlab-runner helm version
```

**Helm output:**

```text
Version: v4.3.0
```

## 3. Fixed Kubernetes Kubeconfig Permission Issues

Initially, GitLab Runner could not access the Kubernetes configuration and Minikube certificates.

### Problem

The original kubeconfig referenced certificate files inside:

```text
/home/teja/.minikube/
```

The `gitlab-runner` Linux user did not have permission to access those files.

We inspected the directory permissions:

```bash
ls -ld /etc/gitlab-runner
```

The directory had `700` permissions and was owned by root.

### Solution

Allowed directory traversal:

```bash
sudo chmod 711 /etc/gitlab-runner
```

Copied the required certificates:

```bash
sudo cp /home/teja/.minikube/profiles/devops-fresh/client.crt /etc/gitlab-runner/.kube/

sudo cp /home/teja/.minikube/profiles/devops-fresh/client.key /etc/gitlab-runner/.kube/

sudo cp /home/teja/.minikube/ca.crt /etc/gitlab-runner/.kube/
```

Updated ownership and permissions:

```bash
sudo chown gitlab-runner:gitlab-runner \
/etc/gitlab-runner/.kube/client.crt \
/etc/gitlab-runner/.kube/client.key \
/etc/gitlab-runner/.kube/ca.crt

sudo chmod 600 \
/etc/gitlab-runner/.kube/client.crt \
/etc/gitlab-runner/.kube/client.key \
/etc/gitlab-runner/.kube/ca.crt
```

Updated the certificate paths in the kubeconfig:

```bash
sudo sed -i \
-e 's|/home/teja/.minikube/profiles/devops-fresh/client.crt|/etc/gitlab-runner/.kube/client.crt|g' \
-e 's|/home/teja/.minikube/profiles/devops-fresh/client.key|/etc/gitlab-runner/.kube/client.key|g' \
-e 's|/home/teja/.minikube/ca.crt|/etc/gitlab-runner/.kube/ca.crt|g' \
/etc/gitlab-runner/.kube/config
```

### Verification

```bash
sudo -u gitlab-runner kubectl \
--kubeconfig=/etc/gitlab-runner/.kube/config get nodes
```

Output:

```text
NAME          STATUS   ROLES           VERSION
devops-fresh  Ready    control-plane   v1.35.1
```

This confirmed that GitLab Runner could communicate with Minikube.

However, this configuration used personal administrator credentials. We decided to replace them with a restricted Kubernetes ServiceAccount.

## 4. Created a Dedicated Kubernetes ServiceAccount

### What is a ServiceAccount?

A ServiceAccount is an identity used by applications and automation tools to communicate with the Kubernetes API.

Instead of allowing GitLab Runner to use personal administrator credentials, we created a dedicated identity.

**ServiceAccount name:** `gitlab-deployer`  
**Namespace:** `junior-devops`

Command:

```bash
kubectl create serviceaccount gitlab-deployer \
-n junior-devops
```

Verification:

```bash
kubectl get serviceaccount gitlab-deployer \
-n junior-devops
```

The ServiceAccount was created successfully.

## 5. Created a Kubernetes Role

### What is a Role?

A Kubernetes Role defines the permissions that an identity can have within a particular namespace.

We inspected the existing application resources:

```bash
kubectl get deployments,services,hpa,configmaps,secrets \
-n junior-devops
```

Based on the resources required for deployment, we created:

```text
k8s/gitlab-deployer-role.yaml
```

### Role configuration

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: gitlab-deployer-role
  namespace: junior-devops

rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  - apiGroups: [""]
    resources: ["services", "configmaps", "secrets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  - apiGroups: ["autoscaling"]
    resources: ["horizontalpodautoscalers"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

Applied the Role:

```bash
kubectl apply -f k8s/gitlab-deployer-role.yaml
```

### Added additional read-only permissions

Helm and deployment verification may need to inspect Pods, Events and ReplicaSets.

We added read-only access to these resources:

```bash
kubectl patch role gitlab-deployer-role -n junior-devops \
--type=json \
-p='[{"op":"add","path":"/rules/-","value":{"apiGroups":[""],"resources":["pods","events"],"verbs":["get","list","watch"]}},{"op":"add","path":"/rules/-","value":{"apiGroups":["apps"],"resources":["replicasets"],"verbs":["get","list","watch"]}}]'
```

## 6. Created a Kubernetes RoleBinding

### What is a RoleBinding?

A RoleBinding connects a Role to a ServiceAccount and grants that identity the permissions defined in the Role.

Created:

```text
k8s/gitlab-deployer-rolebinding.yaml
```

Configuration:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: gitlab-deployer-binding
  namespace: junior-devops

subjects:
  - kind: ServiceAccount
    name: gitlab-deployer
    namespace: junior-devops

roleRef:
  kind: Role
  name: gitlab-deployer-role
  apiGroup: rbac.authorization.k8s.io
```

Applied:

```bash
kubectl apply -f k8s/gitlab-deployer-rolebinding.yaml
```

Now the ServiceAccount was connected to the Role.

### Understanding the relationship

```text
ServiceAccount
gitlab-deployer
       |
       |
       v
RoleBinding
gitlab-deployer-binding
       |
       |
       v
Role
gitlab-deployer-role
       |
       |
       v
Permissions inside junior-devops namespace
```

## 7. Verified Kubernetes RBAC Permissions

We used `kubectl auth can-i` to verify whether the ServiceAccount had the expected permissions.

### Check 1: Create Deployments

```bash
kubectl auth can-i create deployments \
--as=system:serviceaccount:junior-devops:gitlab-deployer \
-n junior-devops
```

Output:

```text
yes
```

### Check 2: Update Deployments

```bash
kubectl auth can-i update deployments \
--as=system:serviceaccount:junior-devops:gitlab-deployer \
-n junior-devops
```

Output:

```text
yes
```

### Check 3: Create Services

```bash
kubectl auth can-i create services \
--as=system:serviceaccount:junior-devops:gitlab-deployer \
-n junior-devops
```

Output:

```text
yes
```

### Check 4: Access another namespace

```bash
kubectl auth can-i create deployments \
--as=system:serviceaccount:junior-devops:gitlab-deployer \
-n default
```

Output:

```text
no
```

### Check 5: List Pods

```bash
kubectl auth can-i list pods \
--as=system:serviceaccount:junior-devops:gitlab-deployer \
-n junior-devops
```

Output:

```text
yes
```

**Result:** The ServiceAccount has the required permissions in `junior-devops`, but does not have permission to create Deployments in the `default` namespace.

This demonstrates namespace-scoped access control using Kubernetes RBAC.

## 8. Generated a ServiceAccount Token

A ServiceAccount token is used to authenticate with the Kubernetes API.

Generated a temporary token:

```bash
kubectl create token gitlab-deployer \
-n junior-devops \
--duration=24h
```

The command succeeded.

The token is temporary and expires after 24 hours.

**Security:** The token should never be committed to Git or shared publicly.

## 9. Created and Tested a Restricted Kubeconfig

We created a separate kubeconfig for the new ServiceAccount.

File location:

```text
/tmp/gitlab-deployer-config
```

The kubeconfig was configured with:

- Kubernetes API server address from the current Minikube context.
- Minikube CA certificate.
- `gitlab-deployer` ServiceAccount token.
- Dedicated Kubernetes context.
- Default namespace set to `junior-devops`.

### Tested access to the project namespace

```bash
kubectl --kubeconfig=/tmp/gitlab-deployer-config \
get pods -n junior-devops
```

Output showed all seven Pods running:

```text
junior-devops-backend       2 Pods Running
junior-devops-frontend      2 Pods Running
todo-postgres               3 Pods Running
```

### Tested access to the default namespace

```bash
kubectl --kubeconfig=/tmp/gitlab-deployer-config \
get pods -n default
```

Output:

```text
Error from server (Forbidden):
pods is forbidden
```

**Result:** Both tests behaved as expected. The restricted kubeconfig successfully authenticates as `gitlab-deployer` and Kubernetes enforces the namespace permissions.

---

## 10. Today's Achievements

| Task | Status |
|---|---|
| Verified Docker access for GitLab Runner | Completed |
| Verified kubectl access | Completed |
| Verified Helm access | Completed |
| Fixed kubeconfig certificate permissions | Completed |
| Created dedicated ServiceAccount | Completed |
| Created Kubernetes Role | Completed |
| Created Kubernetes RoleBinding | Completed |
| Verified namespace-scoped RBAC | Completed |
| Generated temporary ServiceAccount token | Completed |
| Created restricted kubeconfig | Completed |
| Tested restricted namespace access | Completed |
| Installed restricted kubeconfig for GitLab Runner | Pending |

---

## 11. Pending Work – Next Session

We stopped after successfully testing the restricted kubeconfig.

### Step 1: Install the restricted kubeconfig

Copy the tested kubeconfig to the GitLab Runner directory:

```bash
sudo cp /tmp/gitlab-deployer-config \
/etc/gitlab-runner/.kube/config

sudo chown gitlab-runner:gitlab-runner \
/etc/gitlab-runner/.kube/config

sudo chmod 600 \
/etc/gitlab-runner/.kube/config
```

Then verify:

```bash
sudo -u gitlab-runner kubectl \
--kubeconfig=/etc/gitlab-runner/.kube/config \
get pods -n junior-devops
```

Also verify that access to the `default` namespace is denied.

**Note:** The token is valid for only 24 hours. Generate a fresh token if it has expired.

### Step 2: Prepare Helm Chart

Organize our application Kubernetes resources into a Helm chart.

### Step 3: Configure GitLab Container Registry

Configure the registry to store backend and frontend Docker images.

### Step 4: Create GitLab CI/CD Pipeline

Create:

```text
.gitlab-ci.yml
```

Planned stages:

1. Test
2. Build
3. Push
4. Deploy

### Step 5: Execute the pipeline

Push a code change to GitLab and verify that the pipeline automatically builds, pushes and deploys the updated application.

---

## 12. Important Security Notes

1. The temporary ServiceAccount token expires after 24 hours.
2. The current Role grants access to Secrets in the `junior-devops` namespace. This includes database credentials, so we should review whether these permissions can be narrowed.
3. The GitLab Runner registration token was exposed earlier and must be rotated before running CI/CD jobs.
4. Never commit kubeconfig files, authentication tokens, database passwords or other sensitive credentials to Git.
5. Keep GitLab Runner restricted to the trusted project.

---

## 13. Important Concepts Learned Today

| Concept | Explanation |
|---|---|
| GitLab Runner | Executes CI/CD pipeline jobs |
| Kubeconfig | Contains Kubernetes connection and authentication configuration |
| ServiceAccount | Identity used by automation tools and applications |
| Role | Defines namespace-level permissions |
| RoleBinding | Connects a Role to an identity |
| RBAC | Role-Based Access Control |
| ServiceAccount Token | Authenticates the ServiceAccount with Kubernetes |
| Namespace-scoped permissions | Restricts access to a particular namespace |

### Key Learning

**Principle of Least Privilege:** An automation tool should receive only the permissions required to perform its job.

We applied this principle by creating a dedicated Kubernetes ServiceAccount for GitLab Runner instead of using personal administrator credentials.

---

**End of Day 4 – November 26, 2026**

**Next session:** Complete the restricted GitLab Runner Kubernetes authentication setup and begin Helm Chart preparation.
