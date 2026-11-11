
# Junior DevOps Engineering Assignment
## Highly Available Three-Tier Application on Kubernetes

**Project Start Date:** November 5, 2026  
**Current Progress Date:** November 10, 2026  
**Project Deadline:** December 4, 2026  
**Current Phase:** Week 1 – Application Development and Dockerization

---

# 1. Project Overview

The objective of this project is to migrate a small internal application into a highly available, containerized application running on Kubernetes.

The main goal is to understand how modern DevOps practices help organizations deploy applications reliably, minimize downtime, handle failures, scale workloads, and monitor application performance.

Instead of running the application as a single application on one machine, we are separating it into three major components:

1. Frontend – User interface.
2. Backend – Application logic and API.
3. Database – Persistent data storage.

We will eventually deploy these components using Kubernetes and implement high availability, automated deployment, monitoring, and failure testing.

## 1.1 Project Objectives

The major objectives are:

- Develop a three-tier application.
- Containerize the frontend and backend using Docker.
- Configure multiple application services using Docker Compose.
- Deploy the application to Kubernetes.
- Configure Kubernetes Deployments and Services.
- Implement application scalability using multiple replicas and Horizontal Pod Autoscaler (HPA).
- Configure a highly available PostgreSQL database using CloudNativePG.
- Implement CI/CD using GitLab CI.
- Configure monitoring using Prometheus and Grafana.
- Perform load testing using k6 or Locust.
- Test application availability during pod, node, and database failures.

---

# 2. Project Timeline

The assignment is divided into four weeks.

| Week | Dates | Planned Activities |
|---|---|---|
| Week 1 | Nov 5 – Nov 11 | Application development, Docker, Kubernetes basics |
| Week 2 | Nov 12 – Nov 18 | CI/CD and CloudNativePG database setup |
| Week 3 | Nov 19 – Nov 25 | Monitoring, scaling, and performance testing |
| Week 4 | Nov 26 – Dec 4 | Failure testing, documentation, and final demonstration |

Our current focus is Week 1.

---

# 3. Development Environment

We selected a local development environment to build and test the application before deploying it to Kubernetes.

## 3.1 Tools and Technologies

| Technology | Purpose |
|---|---|
| Windows | Host operating system |
| WSL Ubuntu | Linux development environment |
| VS Code | Code editor |
| Python 3.12.3 | Backend development |
| Flask | Backend API framework |
| PostgreSQL 16 | Relational database |
| Docker Desktop | Container management |
| Docker | Application containerization |
| Docker Compose | Multi-container application management |
| Nginx Alpine | Frontend web server |
| Minikube | Local Kubernetes cluster |
| Helm | Kubernetes package management |
| Git and GitHub | Source code management |

## 3.2 Laptop Resource Considerations

The development environment has approximately 8 GB of physical RAM.

To avoid unnecessary resource consumption:

- We use lightweight Docker images.
- We configured CPU and memory limits for our application containers.
- We reused the existing PostgreSQL container instead of creating another database.
- We kept unrelated projects and containers untouched.
- We plan to use the existing Minikube cluster instead of installing another Kubernetes environment.

---

# 4. Day 1 – November 5, 2026
## Project Understanding and Initial Planning

During the first day, we focused on understanding the assignment requirements and identifying the architecture needed to complete the project.

### Activities

1. Reviewed the assignment objectives.
2. Understood the concept of a three-tier application.
3. Identified the technologies required for implementation.
4. Planned the application architecture.
5. Divided the assignment into four weekly milestones.

### Understanding the Three-Tier Architecture

A three-tier application separates responsibilities into different layers.

**Frontend:**

The frontend is the user-facing component. Users interact with it through a browser.

**Backend:**

The backend receives requests from the frontend, processes application logic, and communicates with the database.

**Database:**

The database stores application information permanently.

### Planned Application

We selected a Todo application for implementation.

The application allows users to:

- Add Todo tasks.
- View existing tasks.
- Update task completion status.
- Delete tasks.

This application provides a simple way to understand containerization, networking, persistent storage, Kubernetes deployment, and application availability.

---

# 5. Day 2 – November 6, 2026
## Application Structure and Backend Preparation

We prepared the project workspace and started organizing the application components.

### Project Workspace

The project directory is:

`~/junior-devops-ha-app`

The initial project structure was organized into separate directories.

```text
junior-devops-ha-app/
│
├── backend/
│   └── app.py
│
├── frontend/
│   └── index.html
│
├── database/
│   └── init.sql
│
├── docker/
│
├── k8s/
│
├── requirements.txt
├── .env
├── .gitignore
└── README.md
```

### Backend Preparation

We selected Python Flask to develop the backend.

The backend is responsible for:

- Handling HTTP requests.
- Processing Todo operations.
- Connecting to PostgreSQL.
- Returning JSON responses.
- Providing a health-check endpoint.

### Database Preparation

We selected PostgreSQL 16 as the relational database.

The database table is named `todos`.

The table contains:

| Column | Description |
|---|---|
| id | Unique Todo identifier |
| task | Todo task description |
| completed | Task completion status |
| created_at | Record creation timestamp |

The SQL initialization script is stored in:

`database/init.sql`

```sql
CREATE TABLE IF NOT EXISTS todos (
    id BIGSERIAL PRIMARY KEY,
    task TEXT NOT NULL,
    completed BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

This script creates the Todo table when the PostgreSQL database is initialized for the first time.

---

# 6. Day 3 – November 7, 2026
## PostgreSQL Container and Backend Integration

During this stage, we configured PostgreSQL in Docker and connected our Flask application to the database.

### 6.1 PostgreSQL Container

We created a separate PostgreSQL container.

**Container name:**

`junior-devops-postgres`

**Image:**

`postgres:16-alpine`

**Database name:**

`todo_db`

**Database user:**

`todo_user`

**Internal PostgreSQL port:**

`5432`

**Host port:**

`5433`

The database is accessible from the local development environment through:

`localhost:5433`

### 6.2 Persistent Database Storage

We created a Docker named volume:

`junior-devops-postgres-data`

The volume is mounted at:

`/var/lib/postgresql/data`

This ensures database files are stored outside the PostgreSQL container's writable filesystem.

If the PostgreSQL container is restarted, the data remains available through the persistent volume.

### 6.3 Database Initialization

We mounted the SQL initialization script into:

`/docker-entrypoint-initdb.d/01-init.sql`

PostgreSQL executed the initialization script during the initial database setup.

The logs confirmed that the database was ready and the table creation command succeeded.

### 6.4 Backend Database Connection

We configured Flask to use environment variables for database connectivity.

The backend reads:

- DB_HOST
- DB_PORT
- DB_NAME
- DB_USER
- DB_PASSWORD

We used `python-dotenv` to load local environment variables.

The `.env` file is excluded from Git using `.gitignore`.

**Important:** Database passwords and other secrets must not be committed to the Git repository.

### 6.5 Database Connectivity

We tested the connection between Flask and PostgreSQL.

The backend successfully connected to the database.

We also verified that the health endpoint could execute a database query.

---

# 7. Day 4 – November 8, 2026
## Backend API and Frontend Development

During this stage, we implemented the main Todo application functionality.

### 7.1 Backend API Development

We implemented the following API endpoints.

| HTTP Method | Endpoint | Purpose |
|---|---|---|
| GET | `/` | Welcome endpoint |
| GET | `/health` | Application and database health |
| GET | `/todos` | Retrieve all Todo records |
| POST | `/todos` | Create a new Todo |
| PUT | `/todos/<id>` | Update a Todo |
| DELETE | `/todos/<id>` | Delete a Todo |

### 7.2 Health Check

The health endpoint executes a database query:

```sql
SELECT 1;
```

If the database connection succeeds, the backend returns a healthy response.

If the database connection fails, the endpoint returns HTTP 503.

### 7.3 CRUD Operations

CRUD stands for:

- Create
- Read
- Update
- Delete

We implemented all four operations using Flask and PostgreSQL.

**Create:** Adds a new Todo record.

**Read:** Retrieves Todo records from PostgreSQL.

**Update:** Changes the task information or completion status.

**Delete:** Removes a Todo record.

### 7.4 Backend Testing

We performed local API testing using curl.

All four CRUD operations were individually tested successfully.

We also performed a Python syntax check:

```bash
python -m py_compile backend/app.py
```

The command completed successfully.

### 7.5 Frontend Development

We created the frontend using HTML, CSS, and JavaScript.

The frontend communicates with the backend using HTTP requests.

The API URL used by the browser is:

```javascript
const API_URL = "http://localhost:5001/todos";
```

We configured the backend CORS policy to allow the frontend origin:

`http://localhost:8081`

We initially encountered a browser CORS issue when using a different localhost address. We resolved it by consistently using `localhost`.

### 7.6 Frontend Verification

We opened the application in a browser:

`http://localhost:8081`

We verified that the application could display Todo records and create new tasks.

---

# 8. Day 5 – November 9, 2026
## Dockerization Preparation and Network Configuration

We prepared the application for containerization.

### 8.1 Docker Network

We created a user-defined Docker network:

`junior-devops-network`

A user-defined bridge network allows containers to communicate using their container names.

We connected the existing PostgreSQL container to this network.

The PostgreSQL container is accessible to other containers through:

`junior-devops-postgres`

### 8.2 Database Connection: Host vs Container

We understood an important difference between host-side and container-side database connections.

**When running Flask locally:**

```text
DB_HOST=localhost
DB_PORT=5433
```

**When running Flask inside Docker:**

```text
DB_HOST=junior-devops-postgres
DB_PORT=5432
```

This is because containers communicate through the Docker network using the database container's name and internal port.

### 8.3 Docker Ignore and Git Ignore

We prepared `.gitignore` to prevent unnecessary files and secrets from entering the Git repository.

```gitignore
.env
.venv/
__pycache__/
*.pyc
```

The `.env` file contains database credentials and must remain private.

---

# 9. Day 6 – November 10, 2026
## Dockerfiles, Docker Compose, Container Deployment and Testing

This was a major implementation milestone.

We successfully containerized the frontend and backend and deployed them using Docker Compose.

### 9.1 Backend Dockerfile

Location:

`backend/Dockerfile`

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY backend/ .

EXPOSE 5001

CMD ["python", "app.py"]
```

**Explanation:**

1. `FROM python:3.12-slim` – Uses a lightweight Python base image.
2. `WORKDIR /app` – Sets the working directory.
3. `COPY requirements.txt .` – Copies the Python dependencies file.
4. `RUN pip install` – Installs application dependencies.
5. `COPY backend/ .` – Copies backend application files.
6. `EXPOSE 5001` – Documents the backend container port.
7. `CMD` – Starts the Flask application.

### 9.2 Frontend Dockerfile

Location:

`frontend/Dockerfile`

```dockerfile
FROM nginx:alpine

COPY frontend/index.html /usr/share/nginx/html/index.html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

**Explanation:**

1. `FROM nginx:alpine` – Uses a lightweight Nginx image.
2. `COPY` – Copies the frontend HTML file into the Nginx web directory.
3. `EXPOSE 80` – Documents the Nginx container port.
4. `CMD` – Starts Nginx in the foreground.

### 9.3 Docker Compose Configuration

We created `docker-compose.yml` in the project root.

The file defines two services:

- Backend
- Frontend

The existing PostgreSQL container is reused rather than recreated.

The configuration includes:

- Build configuration.
- Container names.
- Port mappings.
- Database environment variables.
- Docker network configuration.
- Restart policies.
- CPU and memory limits.

The application services are configured with:

| Service | Host Port | Container Port |
|---|---:|---:|
| Backend | 5001 | 5001 |
| Frontend | 8081 | 80 |

The ports are bound to `127.0.0.1`, so the services are exposed only through the local machine's loopback interface.

### 9.4 Docker Compose Network

Both application containers use the existing external network:

`junior-devops-network`

The database container is already connected to this network.

This allows the backend to connect to PostgreSQL using its container name.

### 9.5 Environment Variables

Docker Compose reads the `.env` file in the project root for variable substitution.

The backend receives:

```yaml
environment:
  DB_HOST: junior-devops-postgres
  DB_PORT: "5432"
  DB_NAME: todo_db
  DB_USER: todo_user
  DB_PASSWORD: ${DB_PASSWORD}
```

We verified that `DB_PASSWORD` was set without displaying its actual value.

### 9.6 Docker Compose Validation

Before building the application, we validated the configuration.

```bash
docker compose config --quiet
```

This checks the Compose configuration without starting the services or printing the resolved secrets.

### 9.7 Backend Image Build

Command:

```bash
docker compose build backend
```

Result:

**Successfully built:**

`junior-devops-ha-app-backend:latest`

### 9.8 Frontend Image Build

Command:

```bash
docker compose build frontend
```

Result:

**Successfully built:**

`junior-devops-ha-app-frontend:latest`

We verified both images using:

```bash
docker images | grep junior-devops-ha-app
```

The output confirmed:

| Image | Image ID | Reported Disk Usage |
|---|---|---:|
| junior-devops-ha-app-backend | d04f40579f15 | 227 MB |
| junior-devops-ha-app-frontend | 6881780a0bb1 | 93.7 MB |

### 9.9 Starting Application Containers

We started the application using:

```bash
docker compose up -d
```

Both containers started successfully.

Container names:

- `junior-devops-backend`
- `junior-devops-frontend`

### 9.10 Container Status Verification

We verified the containers using:

```bash
docker compose ps
```

Both containers showed the `Up` status.

The port mappings were:

```text
Backend:
127.0.0.1:5001 -> 5001

Frontend:
127.0.0.1:8081 -> 80
```

### 9.11 Backend Health Check

We tested the backend using:

```bash
curl http://localhost:5001/health
```

Result:

**Healthy**

The backend successfully connected to PostgreSQL.

### 9.12 Frontend Testing

We opened:

`http://localhost:8081`

We verified that the frontend could communicate with the containerized backend.

We tested:

- Adding Todo tasks.
- Retrieving Todo tasks.
- Updating Todo tasks.
- Deleting Todo tasks.

The backend logs showed successful HTTP responses.

Examples:

```text
GET /health HTTP/1.1 200
GET /todos HTTP/1.1 200
OPTIONS /todos HTTP/1.1 200
POST /todos HTTP/1.1 201
```

The HTTP status codes confirmed that requests were being processed successfully.

### 9.13 Backend Logs

We inspected backend logs using:

```bash
docker compose logs --tail=30 backend
```

The logs confirmed that:

- Flask started successfully.
- The application listened on port 5001.
- Health checks returned HTTP 200.
- Todo retrieval requests returned HTTP 200.
- Todo creation returned HTTP 201.
- Browser CORS preflight requests succeeded.

One observation is that Flask is currently running its built-in development server.

For production deployment, we will replace it with a production WSGI server such as Gunicorn.

### 9.14 Backend Restart and Data Persistence Test

We performed an important reliability test.

First, we created a Todo task through the frontend.

Then we restarted only the backend container:

```bash
docker compose restart backend
```

After restarting the backend, we refreshed the frontend.

**Result:**

The Todo data was still visible.

This confirmed that the data was stored in PostgreSQL rather than being held only in the backend container's memory.

The PostgreSQL container and its named volume remained separate from the backend restart.

This is an important containerization concept: application containers can restart independently while persistent database storage remains available.

---

# 10. Current Application Architecture

Our current Docker-based architecture is:

```text
                  USER
                    |
                    |
               WEB BROWSER
                    |
                    |
             localhost:8081
                    |
                    v
          +-------------------+
          | FRONTEND CONTAINER|
          |                   |
          |     NGINX         |
          |    Port 80        |
          +-------------------+
                    |
                    |
             HTTP API Requests
                    |
                    v
             localhost:5001
                    |
                    v
          +-------------------+
          | BACKEND CONTAINER |
          |                   |
          |   Python Flask    |
          |    Port 5001      |
          +-------------------+
                    |
                    |
             Docker Network
          junior-devops-network
                    |
                    v
          +-------------------+
          | DATABASE CONTAINER|
          |                   |
          |   PostgreSQL 16   |
          |    Port 5432      |
          +-------------------+
                    |
                    |
                    v
          +-------------------+
          |  NAMED VOLUME     |
          |                   |
          | Persistent Data   |
          +-------------------+
```

### Important Architecture Details

The frontend communicates with the backend through the browser using the host-mapped backend URL.

The backend communicates with PostgreSQL using the Docker network and database container name.

The database stores its data in a Docker named volume.

The database is not currently managed by Docker Compose. It is an existing, separately managed container that we intentionally reused.

---

# 11. Current Project Directory Structure

The project currently has the following structure:

```text
junior-devops-ha-app/
│
├── backend/
│   ├── app.py
│   ├── app.py.backup
│   └── Dockerfile
│
├── frontend/
│   ├── index.html
│   └── Dockerfile
│
├── database/
│   └── init.sql
│
├── docker/
│
├── k8s/
│
├── .env
├── .gitignore
├── .venv/
├── requirements.txt
├── docker-compose.yml
└── README.md
```

**Important notes:**

- `backend/app.py` contains the current Flask backend.
- `backend/app.py.backup` preserves the earlier implementation.
- `.env` contains local database credentials and must not be committed.
- `.venv/` contains the local Python virtual environment and must not be committed.
- `docker/` and `k8s/` are currently available for future project files.

---

# 12. Current Project Status

## Week 1 – Docker Implementation

| Task | Status |
|---|---|
| Project requirements understood | Completed |
| Three-tier architecture planned | Completed |
| Project workspace prepared | Completed |
| Flask backend developed | Completed |
| PostgreSQL database configured | Completed |
| CRUD API endpoints implemented | Completed |
| Frontend developed | Completed |
| Backend database integration | Completed |
| Docker network configured | Completed |
| Backend Dockerfile created | Completed |
| Frontend Dockerfile created | Completed |
| Docker Compose configured | Completed |
| Backend Docker image built | Completed |
| Frontend Docker image built | Completed |
| Application containers started | Completed |
| Backend health check tested | Completed |
| Frontend integration tested | Completed |
| CRUD operations tested through frontend | Completed |
| Backend restart persistence tested | Completed |

## Kubernetes Implementation

The following tasks are still pending:

| Task | Status |
|---|---|
| Kubernetes Deployment YAML for backend | Pending |
| Kubernetes Deployment YAML for frontend | Pending |
| Kubernetes Services | Pending |
| Application deployment to Minikube | Pending |
| Application testing inside Kubernetes | Pending |
| Kubernetes health probes | Pending |
| Backend replica configuration | Pending |

**Current conclusion:** The Docker portion of Week 1 is completed. The entire Week 1 milestone is not yet complete because Kubernetes basics and deployment remain to be implemented.

---

# 13. Remaining Project Roadmap

## Week 1 – November 5 to November 11
### Application and Kubernetes Basics

Remaining work:

1. Understand Kubernetes architecture.
2. Create backend Deployment YAML.
3. Create frontend Deployment YAML.
4. Create Kubernetes Services.
5. Deploy the application to the existing Minikube cluster.
6. Verify frontend-to-backend communication inside Kubernetes.
7. Test application availability.

## Week 2 – November 12 to November 18
### CI/CD and Highly Available Database

Planned work:

1. Configure GitLab CI.
2. Automate application testing.
3. Build Docker images through the pipeline.
4. Publish images to a container registry.
5. Automate Kubernetes deployment.
6. Install and configure CloudNativePG.
7. Configure PostgreSQL primary and two replicas.
8. Understand database replication and failover.

## Week 3 – November 19 to November 25
### Monitoring and Scaling

Planned work:

1. Install metrics-server.
2. Configure Horizontal Pod Autoscaler.
3. Configure Prometheus.
4. Configure Grafana.
5. Monitor application metrics.
6. Perform load testing using k6 or Locust.
7. Observe application behavior under increased traffic.

## Week 4 – November 26 to December 4
### High Availability, Failure Testing and Documentation

Planned work:

1. Test backend pod failures.
2. Test pod restart and recovery.
3. Test node failure scenarios where the environment supports them.
4. Test database failover.
5. Verify application availability during failures.
6. Complete project documentation.
7. Prepare the architecture diagram.
8. Prepare final demonstration.

---

# 14. Important DevOps Concepts Learned So Far

During the implementation, we learned and applied several important concepts.

### 14.1 Containerization

Containerization packages an application and its dependencies into an isolated environment.

We created separate Docker images for the frontend and backend.

### 14.2 Docker Images and Containers

A Docker image is the packaged application template.

A container is a running instance created from that image.

### 14.3 Docker Compose

Docker Compose allows us to define and manage multiple related containers through one YAML file.

We used it to start and manage the frontend and backend.

### 14.4 Docker Networking

Containers connected to a user-defined network can communicate using container names.

We used this feature to connect Flask to PostgreSQL.

### 14.5 Persistent Volumes

Containers are replaceable, but application data should survive container restarts.

We used a Docker named volume for PostgreSQL storage.

### 14.6 Environment Variables

We separated configuration and credentials from application source code using environment variables.

### 14.7 Health Checks

The `/health` endpoint checks whether the backend can successfully communicate with PostgreSQL.

### 14.8 Application Reliability

We restarted the backend container and verified that the data remained available.

This helped us understand the separation between application runtime and persistent database storage.

---

# 15. Important Commands Used

## Python and Backend

```bash
python -m py_compile backend/app.py
```

## Docker Images

```bash
docker images
docker images | grep junior-devops-ha-app
```

## Docker Containers

```bash
docker ps
docker compose ps
```

## Docker Compose Validation

```bash
docker compose config --quiet
```

## Docker Image Builds

```bash
docker compose build backend
docker compose build frontend
```

## Start Application

```bash
docker compose up -d
```

## Backend Health Check

```bash
curl http://localhost:5001/health
```

## Backend Logs

```bash
docker compose logs --tail=30 backend
```

## Restart Backend

```bash
docker compose restart backend
```

## Database and Docker Network

```bash
docker network ls
docker network inspect junior-devops-network
```

---

# 16. Current Running Application Details

| Component | Name | Port |
|---|---|---|
| Frontend container | junior-devops-frontend | 8081 |
| Backend container | junior-devops-backend | 5001 |
| PostgreSQL container | junior-devops-postgres | 5433 on host, 5432 internally |
| Docker network | junior-devops-network | Internal bridge network |
| PostgreSQL volume | junior-devops-postgres-data | Persistent storage |

### Application URLs

Frontend:

`http://localhost:8081`

Backend health:

`http://localhost:5001/health`

Backend Todo API:

`http://localhost:5001/todos`

---

# 17. Important Notes for the Next Development Session

Before starting Kubernetes work:

1. Preserve the existing Docker Compose application.
2. Do not delete or recreate the PostgreSQL container.
3. Do not delete the PostgreSQL named volume.
4. Do not modify unrelated E-Commerce containers.
5. Reuse the existing Minikube installation and cluster where appropriate.
6. Check available laptop memory before starting additional services.
7. Continue using the existing project directory.
8. Keep `.env` and other secrets out of Git.

### Next Immediate Task

Start the Kubernetes implementation.

First, inspect the existing Minikube cluster and Kubernetes context.

Then create the Kubernetes YAML files inside:

`k8s/`

The first files will be:

- Backend Deployment YAML.
- Frontend Deployment YAML.
- Backend Service YAML.
- Frontend Service YAML.

We will implement and test each component incrementally.

---

# 18. Final Progress Summary

As of November 10, 2026, we have successfully developed and containerized a three-tier Todo application.

The Flask backend supports CRUD operations and communicates with PostgreSQL.

The frontend is served using Nginx.

Both frontend and backend have been packaged into Docker images and started using Docker Compose.

The application has been tested through the browser, its backend health endpoint, and application logs.

We also restarted the backend container and verified that Todo records remained available because PostgreSQL data is stored independently in a persistent Docker volume.

**Completed milestone:** Application development, Dockerfiles, Docker Compose, container deployment, integration testing, and database persistence verification.

**Next milestone:** Kubernetes deployment using Minikube.

**Overall project status:** Week 1 Docker implementation completed; Kubernetes implementation pending.

---

**Project:** Highly Available Three-Tier Application on Kubernetes  
**Last Updated:** November 10, 2026

# Junior DevOps Engineering Assignment
## Week 1 – Daily Progress Report
**Date:** November 11, 2026  
**Project:** Highly Available Three-Tier Application on Kubernetes  
**Application:** Junior DevOps Todo Application  
**Status:** Frontend and Backend Kubernetes Deployment Completed – Database Connectivity Testing Pending

---

# 1. Objective

The objective of today's work was to continue migrating the Todo application from Docker Compose to Kubernetes using Minikube.

The main tasks were:

- Configure the frontend for Kubernetes communication.
- Configure Nginx as a reverse proxy.
- Build an updated frontend Docker image.
- Load the image into Minikube.
- Create the frontend Kubernetes Deployment.
- Create the frontend Kubernetes Service.
- Access the application through the Minikube cluster.
- Verify the frontend and backend Pods.

# 2. Development Environment

The following tools and technologies were used:

| Technology | Purpose |
|---|---|
| Windows | Host operating system |
| WSL Ubuntu 24.04 | Linux development environment |
| Docker Desktop | Containerization |
| Docker Engine 29.2.1 | Docker image building and containers |
| Minikube 1.38.1 | Local Kubernetes cluster |
| Kubernetes 1.35.1 | Container orchestration |
| Helm 4.3.0 | Kubernetes package management |
| Python 3.12.3 | Backend development |
| Flask | Backend REST API |
| PostgreSQL 16 | Database |
| Nginx Alpine | Frontend web server and reverse proxy |
| kubectl | Kubernetes resource management |

**Project working directory:**

`/home/teja/junior-devops-ha-app`

**Minikube profile:** `devops-fresh`

**Kubernetes namespace:** `junior-devops`

# 3. Previous Work and Initial Setup

Before today's Kubernetes frontend work, the following components had already been completed.

### 3.1 Application Development

The application consists of:

**Frontend:**
- HTML
- CSS
- JavaScript
- Todo task management interface

**Backend:**
- Python Flask application
- REST API endpoints
- PostgreSQL database integration
- Health-check endpoint

### 3.2 Backend API Endpoints

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/` | Backend welcome response |
| GET | `/health` | Backend and database health check |
| GET | `/todos` | Retrieve Todo tasks |
| POST | `/todos` | Create a Todo task |
| PUT | `/todos/<id>` | Update a Todo task |
| DELETE | `/todos/<id>` | Delete a Todo task |

### 3.3 Docker Compose Setup

The application was successfully containerized using Docker.

Created containers:

- `junior-devops-backend`
- `junior-devops-frontend`
- `junior-devops-postgres`

The existing PostgreSQL container was reused instead of creating another database container.

The frontend and backend were tested using Docker Compose, and the application was able to perform Todo operations.

Database persistence was also tested by restarting the backend container and checking whether existing tasks remained available.

### 3.4 Docker Images

Previously created images:

- `junior-devops-ha-app-backend:latest`
- `junior-devops-ha-app-frontend:latest`

Both images were loaded into the Minikube environment before deploying the backend.

# 4. Kubernetes Cluster Preparation

### 4.1 Starting Minikube

The existing Minikube profile was started:

```bash
minikube start -p devops-fresh
```

Verified the Kubernetes node:

```bash
kubectl get nodes
```

The node was successfully displayed with status:

```text
devops-fresh   Ready   control-plane
```

### 4.2 Creating the Kubernetes Namespace

Created a separate namespace for the assignment:

```bash
kubectl create namespace junior-devops
```

Set the current Kubernetes context to use this namespace:

```bash
kubectl config set-context --current --namespace=junior-devops
```

Verified the namespace:

```bash
kubectl get namespaces
```

The `junior-devops` namespace was successfully created.

### 4.3 Creating the Database Secret

The database environment variables were stored in the existing `.env` file.

Created a Kubernetes Secret:

```bash
kubectl create secret generic junior-devops-db-secret \
  --namespace=junior-devops \
  --from-env-file=.env
```

Verified the Secret:

```bash
kubectl describe secret junior-devops-db-secret -n junior-devops
```

The Secret contained the following keys:

- DB_HOST
- DB_PORT
- DB_NAME
- DB_USER
- DB_PASSWORD

**Security note:** The `.env` file is excluded from Git using `.gitignore`. Database credentials must not be committed to GitHub.

# 5. Backend Kubernetes Deployment

### 5.1 Backend Deployment YAML

Created:

`k8s/backend-deployment.yaml`

The Deployment configuration includes:

- Deployment name: `junior-devops-backend`
- Namespace: `junior-devops`
- Initial replicas: 1
- Container image: `junior-devops-ha-app-backend:latest`
- Container port: 5001
- Image pull policy: Never
- Database password loaded from the Kubernetes Secret.

The database configuration is provided through environment variables.

### 5.2 Backend Deployment Validation

Validated the YAML:

```bash
kubectl apply --dry-run=client -f k8s/backend-deployment.yaml
```

Applied the Deployment:

```bash
kubectl apply -f k8s/backend-deployment.yaml
```

### 5.3 Backend Pod Verification

```bash
kubectl get pods -n junior-devops
```

The backend Pod successfully reached:

```text
READY: 1/1
STATUS: Running
RESTARTS: 0
```

Checked backend logs:

```bash
kubectl logs deployment/junior-devops-backend -n junior-devops
```

Flask started successfully and listened on:

```text
0.0.0.0:5001
```

# 6. Backend Kubernetes Service

Created:

`k8s/backend-service.yaml`

The Service configuration:

| Property | Value |
|---|---|
| Service name | junior-devops-backend |
| Type | ClusterIP |
| Namespace | junior-devops |
| Service port | 5001 |
| Target port | 5001 |

Validated:

```bash
kubectl apply --dry-run=client -f k8s/backend-service.yaml
```

Applied:

```bash
kubectl apply -f k8s/backend-service.yaml
```

Verified:

```bash
kubectl get services -n junior-devops
```

The backend Service was successfully created.

It provides a stable internal Kubernetes endpoint for the backend application.

# 7. Frontend Configuration for Kubernetes

This was one of the main activities completed today.

### 7.1 Identifying the Frontend Networking Problem

The original frontend JavaScript contained:

```javascript
const API_URL = "http://localhost:5001/todos";
```

This configuration worked with Docker Compose.

However, Kubernetes uses internal Service DNS to communicate between Pods.

The browser cannot directly resolve Kubernetes internal Service names because JavaScript executes in the browser rather than inside the frontend Pod.

Therefore, we needed a reverse proxy.

### 7.2 Updating the Frontend API URL

Modified:

`frontend/index.html`

Changed the API URL to:

```javascript
const API_URL = "/todos";
```

This makes the browser send API requests to the same origin that serves the frontend.

### 7.3 Creating the Nginx Configuration

Created:

`frontend/nginx.conf`

Configuration:

```nginx
server {
    listen 80;

    location / {
        root /usr/share/nginx/html;
        index index.html;
        try_files $uri $uri/ /index.html;
    }

    location /todos {
        proxy_pass http://junior-devops-backend:5001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**Purpose:**

- Nginx listens on port 80.
- The root location serves the frontend HTML.
- Requests to `/todos` are forwarded to the backend Kubernetes Service.
- Kubernetes internal DNS resolves `junior-devops-backend`.

This allows the browser to communicate with the backend without directly accessing Kubernetes internal DNS.

# 8. Updating the Frontend Dockerfile

Modified:

`frontend/Dockerfile`

Final configuration:

```dockerfile
FROM nginx:alpine

COPY frontend/index.html /usr/share/nginx/html/index.html

COPY frontend/nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

The Dockerfile now copies:

1. Frontend HTML file.
2. Custom Nginx reverse proxy configuration.

# 9. Building the Updated Frontend Docker Image

Built a new image with a separate version tag:

```bash
docker build -t junior-devops-ha-app-frontend:k8s-v1 -f frontend/Dockerfile .
```

The build completed successfully.

Using a separate tag helps distinguish the Kubernetes-ready frontend image from the original Docker Compose image.

### 9.1 Loading the Image into Minikube

Executed:

```bash
minikube image load junior-devops-ha-app-frontend:k8s-v1 -p devops-fresh
```

Verified:

```bash
minikube image ls -p devops-fresh | grep junior-devops-ha-app-frontend
```

Output confirmed both images:

```text
docker.io/library/junior-devops-ha-app-frontend:latest
docker.io/library/junior-devops-ha-app-frontend:k8s-v1
```

The updated frontend image was successfully loaded into Minikube.

# 10. Frontend Kubernetes Deployment

Created:

`k8s/frontend-deployment.yaml`

Deployment configuration:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: junior-devops-frontend
  namespace: junior-devops

spec:
  replicas: 1

  selector:
    matchLabels:
      app: junior-devops-frontend

  template:
    metadata:
      labels:
        app: junior-devops-frontend

    spec:
      containers:
        - name: frontend
          image: junior-devops-ha-app-frontend:k8s-v1
          imagePullPolicy: Never

          ports:
            - containerPort: 80
              protocol: TCP

          resources:
            requests:
              memory: "32Mi"
              cpu: "50m"
            limits:
              memory: "64Mi"
              cpu: "250m"
```

### 10.1 Deployment Validation

```bash
kubectl apply --dry-run=client -f k8s/frontend-deployment.yaml
```

The dry run completed successfully.

Applied the Deployment:

```bash
kubectl apply -f k8s/frontend-deployment.yaml
```

### 10.2 Frontend Pod Verification

```bash
kubectl get pods -n junior-devops
```

Verified output:

```text
NAME                                      READY   STATUS
junior-devops-backend-5bbff46786-wxwvk   1/1     Running
junior-devops-frontend-7f6c6868c6-l8mqx  1/1     Running
```

Both Pods reached Running status with zero restarts at the time of verification.

# 11. Frontend Kubernetes Service

Created:

`k8s/frontend-service.yaml`

Configuration:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: junior-devops-frontend
  namespace: junior-devops

spec:
  type: NodePort

  selector:
    app: junior-devops-frontend

  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080
```

### 11.1 Service Explanation

| Property | Value | Purpose |
|---|---|---|
| Service name | junior-devops-frontend | Frontend Service |
| Type | NodePort | External access to frontend |
| Port | 80 | Service port |
| Target port | 80 | Nginx container port |
| NodePort | 30080 | Node access port |

Applied the Service:

```bash
kubectl apply -f k8s/frontend-service.yaml
```

Verified:

```bash
kubectl get services -n junior-devops
```

Verified output:

```text
NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)
junior-devops-backend    ClusterIP   10.105.55.213   <none>        5001/TCP
junior-devops-frontend   NodePort    10.96.215.52    <none>        80:30080/TCP
```

Both Kubernetes Services were successfully created.

# 12. Accessing the Frontend Through Minikube

### 12.1 Checking the Minikube IP

Executed:

```bash
minikube ip -p devops-fresh
```

Output:

```text
192.168.58.2
```

Initially attempted to access:

```text
http://192.168.58.2:30080
```

However, the Windows browser displayed:

```text
This site can't be reached
```

### 12.2 Troubleshooting Minikube NodePort Access

Executed:

```bash
minikube service junior-devops-frontend -n junior-devops --url -p devops-fresh
```

Minikube returned:

```text
http://127.0.0.1:41007
```

It also displayed:

```text
Because you are using a Docker driver on linux,
the terminal needs to be open to run it.
```

This is an informational message. Minikube maintains a temporary tunnel while the command is running.

### 12.3 Successful Browser Access

Opened:

```text
http://127.0.0.1:41007
```

**Result: SUCCESS**

The Todo application frontend loaded successfully in the browser through the Minikube Service tunnel.

The terminal running the Minikube service command must remain open while using this temporary URL.

If the tunnel is stopped or the terminal closes, run the command again to obtain an active access URL.

# 13. Current Kubernetes Architecture

The current application architecture is:

```text
                 USER BROWSER
                      |
                      |
             Minikube Service Tunnel
                      |
                      |
              FRONTEND SERVICE
                 NodePort
                  30080
                      |
                      |
              FRONTEND POD
                Nginx :80
                      |
                      |
              Reverse Proxy
                      |
                      |
              BACKEND SERVICE
                 ClusterIP
                  Port 5001
                      |
                      |
               BACKEND POD
                Flask :5001
                      |
                      |
                POSTGRESQL
```

**Important:** The frontend-to-backend reverse proxy is configured, but successful end-to-end API and database operations have not yet been verified in Kubernetes.

# 14. Current Project Status

| Task | Status |
|---|---|
| Flask backend development | Completed |
| Frontend development | Completed |
| PostgreSQL Docker container setup | Completed |
| Docker Compose setup | Completed |
| Docker image creation | Completed |
| Minikube cluster setup | Completed |
| Kubernetes namespace creation | Completed |
| Kubernetes database Secret creation | Completed |
| Backend Deployment | Completed |
| Backend ClusterIP Service | Completed |
| Frontend API URL update | Completed |
| Nginx reverse proxy configuration | Completed |
| Updated frontend Docker image | Completed |
| Frontend image loaded into Minikube | Completed |
| Frontend Deployment | Completed |
| Frontend NodePort Service | Completed |
| Browser access through Minikube tunnel | Verified |
| Kubernetes Todo CRUD testing | Pending |
| Kubernetes backend-to-database connectivity | Pending |
| High availability and scaling | Pending |
| CI/CD pipeline | Pending |
| CloudNativePG deployment | Pending |
| Prometheus and Grafana monitoring | Pending |
| Failure testing | Pending |

# 15. Pending Tasks – Next Working Session

The next session will continue from the current Kubernetes deployment.

### Task 1: Test Todo Operations

Open the application using the active Minikube tunnel.

Test:

- Add a Todo task.
- Retrieve existing tasks.
- Edit a Todo task.
- Mark a task as completed.
- Delete a Todo task.

Check whether each operation works.

### Task 2: Verify Backend Health

Check the backend health endpoint and verify database connectivity.

The backend health endpoint is:

```text
/health
```

It executes a database query and reports whether the database connection is healthy.

### Task 3: Troubleshoot PostgreSQL Connectivity

The existing PostgreSQL container is running separately from the Kubernetes Pods.

Its Docker network connectivity to Minikube Pods has not yet been established or verified.

Investigate this without deleting or recreating the existing PostgreSQL container or its persistent volume.

### Task 4: Complete Week 1

After successful frontend, backend, and database testing:

- Verify application functionality.
- Review Kubernetes Deployments.
- Review Kubernetes Services.
- Confirm application connectivity.
- Update the project documentation.

### Task 5: Continue Week 2

Proceed with:

- GitLab CI/CD pipeline.
- Docker image publishing.
- CloudNativePG installation.
- PostgreSQL primary and replica configuration.
- Persistent storage.

# 16. Important Notes and Safety Considerations

1. Continue using the existing Minikube profile `devops-fresh`.
2. Continue using the existing namespace `junior-devops`.
3. Do not reinstall Docker, Minikube, Kubernetes, or Helm.
4. Do not delete existing Docker containers or PostgreSQL volumes.
5. Do not modify unrelated Kubernetes workloads in the `default` namespace.
6. Keep `.env` excluded from Git.
7. Do not commit database credentials.
8. Keep the terminal running while using the temporary Minikube service tunnel.
9. Do not assume database connectivity merely because the backend Pod is Running.
10. The current deployment uses one replica for each application component; high availability is not yet implemented.

---

# 17. Final Summary

Today's work successfully advanced the Junior DevOps HA Todo Application from Docker Compose deployment to Kubernetes-based frontend and backend deployment.

The backend and frontend Kubernetes Deployments were created successfully, and both Pods reached Running status.

A custom Nginx reverse proxy was configured to forward frontend API requests to the backend Kubernetes Service. The updated frontend Docker image was built, loaded into Minikube, and deployed.

Both Kubernetes Services were created successfully. After troubleshooting the NodePort access issue caused by the local Minikube Docker-driver environment, the application frontend was successfully accessed through the Minikube service tunnel.

**Current stopping point:** The Todo application webpage is accessible through Kubernetes. Complete CRUD functionality and PostgreSQL connectivity still need verification.

**Next session starting point:** Test Todo CRUD operations and verify backend-to-PostgreSQL connectivity without disturbing existing resources.

**Progress:** Frontend and backend Kubernetes deployment completed. Week 1 application integration testing remains in progress.
