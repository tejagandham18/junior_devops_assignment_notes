
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
