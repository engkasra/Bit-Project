# Crypto Exchange Infrastructure Architecture Diagram Specification

## Overview

Create a comprehensive multi-layer architecture diagram showing the complete Crypto Exchange application infrastructure including Kubernetes cluster, external services, CI/CD pipeline, monitoring, and logging stack.

***

## Infrastructure Servers

### Server 1: Kubernetes Master Node

- **IP:** 91.107.191.148
- **Hostname:** Master Node
- **Role:** Kubernetes Control Plane
- **Components:**
    - kube-apiserver (port 6443)
    - kube-controller-manager
    - kube-scheduler
    - etcd (port 2379, 2380)
    - Calico CNI


### Server 2: Kubernetes Worker Node / PostgreSQL Server

- **IP:** 91.107.171.123
- **Hostname:** Worker Node + DB
- **Role:** Kubernetes Worker + Database Server
- **Components:**
    - kubelet
    - kube-proxy
    - Calico CNI
    - PostgreSQL (port 5432) - External, not containerized
    - Container Runtime (containerd/Docker)


### Server 3: ELK Stack Server

- **IP:** 91.107.185.11
- **Hostname:** ELK Server
- **Role:** Centralized Logging
- **Components:**
    - Elasticsearch (port 9200)
    - Kibana (port 5601)

***

## Kubernetes Cluster Components

### Namespace: crypto-exchange

#### Deployment: django-app

- **Replicas:** 2+ (scalable)
- **Container Image:** Docker Hub image
- **Ports:**
    - 8000: Django application (HTTP)
    - 8001: Prometheus metrics endpoint
- **Environment Variables (from ConfigMap/Secret):**
    - DATABASE_HOST: 91.107.171.123
    - DATABASE_PORT: 5432
    - DATABASE_NAME: crypto_exchange
    - DATABASE_USER: crypto_user
    - DATABASE_PASSWORD: (from Secret)
    - DJANGO_SECRET_KEY: (from Secret)


#### Service: django-service

- **Type:** ClusterIP
- **Cluster IP:** (dynamic, e.g., 10.96.x.x)
- **Ports:**
    - Port 80 → TargetPort 8000 (app traffic)
    - Port 8001 → TargetPort 8001 (metrics)
- **Selector:** app=django-app


#### Ingress: crypto-exchange-ingress

- **Ingress Controller:** NGINX Ingress Controller
- **Host:** crypto-exchange.local
- **Rules:**
    - Path: / → django-service:80
- **External Access:** NodePort or LoadBalancer


#### ConfigMap: crypto-exchange-config

- Contains non-sensitive environment variables


#### Secret: crypto-exchange-secret

- Contains:
    - DATABASE_PASSWORD
    - DJANGO_SECRET_KEY


#### CronJob: postgres-daily-backup (optional)

- **Schedule:** 0 2 * * * (daily at 2 AM)
- **Target:** PostgreSQL at 91.107.171.123:5432
- **Storage:** PersistentVolumeClaim

***

### Namespace: monitoring

#### Deployment: prometheus

- **Port:** 9090
- **Purpose:** Metrics collection and storage
- **Scrape Targets:**
    - django-service.crypto-exchange:8001/metrics
    - Kubernetes API metrics
    - Node metrics


#### Service: prometheus-service

- **Type:** NodePort or ClusterIP
- **Port:** 9090


#### Deployment: grafana

- **Port:** 3000
- **NodePort:** 30300 (external access)
- **Data Source:** Prometheus at prometheus-service:9090
- **Credentials:** admin / admin123


#### Service: grafana-service

- **Type:** NodePort
- **Port:** 3000
- **NodePort:** 30300

***

### Namespace: logging

#### DaemonSet: fluent-bit

- **Runs on:** All nodes (master + worker)
- **Purpose:** Log collection from all pods
- **Mounts:**
    - /var/log (host path)
    - /var/lib/docker/containers (host path)
- **Configuration:**
    - Input: Tail /var/log/containers/*.log
    - Filter: Kubernetes metadata enrichment
    - Output: Elasticsearch at 91.107.185.11:9200
- **Authentication:** elastic user
- **Index:** fluentbit-k8s


#### ServiceAccount: fluent-bit

- **RBAC:** ClusterRole with read access to pods, namespaces


#### ConfigMap: fluent-bit-config

- Contains Fluent Bit configuration and parsers

***

### Namespace: ingress-nginx

#### Deployment: ingress-nginx-controller

- **Purpose:** HTTP/HTTPS routing
- **Ports:**
    - 80 (HTTP)
    - 443 (HTTPS)
- **Service Type:** NodePort or LoadBalancer
- **Routes traffic to:** django-service based on hostname

***

## External Services

### PostgreSQL Database

- **Location:** Server 91.107.171.123 (on host, not in container)
- **Port:** 5432
- **Database:** crypto_exchange
- **User:** crypto_user
- **Accessible from:**
    - 91.107.191.148 (Master node)
    - 91.107.185.11 (ELK server)
    - 91.107.171.123 (localhost)
- **Firewall:** UFW rules restricting access to above IPs only
- **Backup:** Daily CronJob dumps to PVC


### Elasticsearch

- **Location:** Server 91.107.185.11
- **Port:** 9200
- **Purpose:** Log storage and search
- **Index:** fluentbit-k8s
- **Authentication:** elastic user with password
- **Security:** xpack.security.enabled=true, SSL disabled for internal use
- **Discovery:** single-node mode
- **Receives logs from:** Fluent Bit DaemonSet in Kubernetes


### Kibana

- **Location:** Server 91.107.185.11
- **Port:** 5601
- **Purpose:** Log visualization and analysis
- **Connects to:** Elasticsearch at localhost:9200
- **Authentication:** kibana_system user (backend), elastic user (web login)
- **Data View:** fluentbit-k8s* index pattern

***

## CI/CD Pipeline

### GitHub Repository

- **Source:** Application code in Exchange/ directory
- **Manifests:** Kubernetes YAML in manifests/v2/
- **Workflow:** .github/workflows/ci-cd.yml


### GitHub Actions Workflow

- **Trigger:** Push to main branch
- **Steps:**

1. Checkout code
2. Build Docker image from Exchange/ directory
3. Push to Docker Hub
4. Configure kubectl with kubeconfig (from GitHub Secret KUBE_CONFIG)
5. Update Kubernetes deployment: kubectl set image
6. Rollout deployment
- **Secrets:**
    - DOCKER_USERNAME
    - DOCKER_PASSWORD
    - KUBE_CONFIG (base64 encoded)


### Docker Hub

- **Registry:** Docker Hub
- **Image:** username/crypto-exchange:latest
- **Pulled by:** Kubernetes worker nodes

***

## Network Flow

### User Traffic Flow

1. **User Browser** → http://crypto-exchange.local:NodePort
2. **Ingress Controller** (NGINX) → Routes based on hostname
3. **django-service** (ClusterIP) → Load balances to Django pods
4. **Django Pods** → Query PostgreSQL at 91.107.171.123:5432
5. **PostgreSQL** → Returns data
6. **Django** → Renders response
7. **User Browser** ← Receives HTTP response

### Monitoring Flow

1. **Django Pods** → Expose /metrics endpoint on port 8001
2. **Prometheus** → Scrapes django-service:8001/metrics every 15s
3. **Grafana** → Queries Prometheus:9090 for metrics
4. **Admin** → Views dashboards at http://NODE_IP:30300

### Logging Flow

1. **All Kubernetes Pods** → Write logs to stdout/stderr
2. **Container Runtime** → Writes logs to /var/log/containers/*.log
3. **Fluent Bit DaemonSet** → Tails log files
4. **Fluent Bit** → Enriches with Kubernetes metadata (pod name, namespace, labels)
5. **Fluent Bit** → Sends to Elasticsearch at 91.107.185.11:9200
6. **Elasticsearch** → Indexes logs in fluentbit-k8s
7. **Kibana** → Queries Elasticsearch for log visualization
8. **Admin** → Views logs at http://91.107.185.11:5601

### CI/CD Flow

1. **Developer** → Commits code to GitHub
2. **GitHub Actions** → Triggered automatically
3. **Docker Build** → Creates new image
4. **Docker Hub** → Stores image
5. **kubectl** → Updates Kubernetes deployment
6. **Kubernetes** → Pulls new image and rolling update
7. **Django Pods** → Updated with new code

***

## Security \& Firewall Rules

### PostgreSQL Server (91.107.171.123)

- **UFW Rules:**
    - Allow from 91.107.191.148 to port 5432 (Master node)
    - Allow from 91.107.185.11 to port 5432 (ELK server)
    - Allow from 91.107.171.123 to port 5432 (localhost)
    - Deny all other traffic to port 5432


### ELK Server (91.107.185.11)

- **UFW Rules:**
    - Allow port 9200 (Elasticsearch) from Kubernetes nodes
    - Allow port 5601 (Kibana) from admin IP/all
    - Allow port 22 (SSH)


### Kubernetes Cluster

- **CNI:** Calico for pod networking
- **Network Policies:** (if configured)
- **Service Mesh:** None (can be added later)

***

## Diagram Requirements

### High-Level View (Layer 1)

- Show 3 physical servers with IPs
- Show external user access
- Show GitHub → CI/CD → Kubernetes flow
- Show monitoring and logging connections


### Mid-Level View (Layer 2)

- Show Kubernetes namespaces as logical groups
- Show services, ingress, deployments
- Show external PostgreSQL and ELK stack
- Show network connections with protocols and ports


### Low-Level View (Layer 3)

- Show individual pods with replicas
- Show ConfigMaps, Secrets, PVCs
- Show all port mappings (8000→8000, 80→8000, etc.)
- Show Fluent Bit on each node
- Show exact IP addresses and DNS names
- Show all environment variables flow
- Show Kubernetes API server interactions


### Data Flow Annotations

- User HTTP requests (red arrows)
- Database queries (blue arrows)
- Metrics scraping (green arrows)
- Log streaming (orange arrows)
- CI/CD deployments (purple arrows)


### Component Details to Include

- Each component with its port number
- Service types (ClusterIP, NodePort, LoadBalancer)
- Persistent storage (PVC)
- Secrets and ConfigMaps
- RBAC roles and service accounts
- Ingress rules and routing

***

## Color Coding Suggestion

- **User-facing components:** Blue
- **Application layer:** Green
- **Data layer:** Orange
- **Monitoring:** Yellow
- **Logging:** Purple
- **CI/CD:** Red
- **External services:** Gray

***

