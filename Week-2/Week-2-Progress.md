
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
