# WordPress on Kubernetes 🚀

A Docker Compose to Kubernetes migration of a WordPress application, deployed with Helm and running on a Minikube cluster hosted on AWS EC2.

The project includes Amazon ECR, NGINX Ingress, persistent storage, health probes, Prometheus, Grafana, and Kubernetes-native application management.

---

## 🏗️ Architecture

```text
                         AWS EC2
                            │
                            ▼
                        Minikube
                            │
                            ▼
                 NGINX Ingress Controller
                            │
                  wordpress.local
                            │
                            ▼
                  WordPress Service
                     │            │
                     ▼            ▼
              WordPress Pod  WordPress Pod
                     │            │
                     └─────┬──────┘
                           │
                           ▼
                  MariaDB Service
                           │
                           ▼
                 MariaDB StatefulSet
                           │
                           ▼
                 PersistentVolumeClaim
                           │
                           ▼
                    Persistent Storage


              ┌─────────────────────────┐
              │       Monitoring        │
              │                         │
              │ Prometheus              │
              │ Grafana                 │
              │ Alertmanager            │
              │ kube-state-metrics      │
              │ Node Exporter           │
              └─────────────────────────┘
```

---

## 🎯 Project Goals

The goal of this project was to migrate a Docker Compose-based WordPress application to Kubernetes while implementing common DevOps and Kubernetes practices:

* Kubernetes Deployments
* StatefulSets
* Persistent storage
* Kubernetes Services
* NGINX Ingress
* Helm
* Amazon ECR
* Health probes
* Resource requests and limits
* Prometheus monitoring
* Grafana dashboards

---

## 🛠️ Technology Stack

| Technology         | Purpose                              |
| ------------------ | ------------------------------------ |
| AWS EC2            | Infrastructure                       |
| Kubernetes         | Container orchestration              |
| Minikube           | Kubernetes cluster                   |
| Helm               | Application packaging and deployment |
| Docker             | Container runtime                    |
| Amazon ECR         | Container image registry             |
| NGINX Ingress      | HTTP routing                         |
| WordPress          | Web application                      |
| MariaDB            | Database                             |
| Prometheus         | Metrics collection                   |
| Grafana            | Metrics visualization                |
| Alertmanager       | Alert management                     |
| kube-state-metrics | Kubernetes metrics                   |
| Node Exporter      | Node metrics                         |

---

# 📦 Application Architecture

## WordPress

WordPress is deployed using a Kubernetes `Deployment` with **2 replicas**.

```text
                    WordPress Deployment
                       /            \
                      /              \
                     ▼                ▼
              WordPress Pod     WordPress Pod
```

The Deployment provides:

* 2 application replicas
* Rolling update capability
* Self-healing
* Readiness probe
* Liveness probe
* Resource requests and limits

---

## Database

MariaDB is deployed using a Kubernetes `StatefulSet`.

```text
              MariaDB StatefulSet
                       │
                       ▼
                  MariaDB Pod
                       │
                       ▼
               Persistent Storage
```

A StatefulSet is used because the database is a stateful workload that requires persistent storage.

---

## Persistent Storage

MariaDB uses a `5Gi` PersistentVolumeClaim.

```text
MariaDB Pod
    │
    ▼
PersistentVolumeClaim
    │
    ▼
Persistent Volume
```

The PVC allows database data to survive Pod restarts.

---

# 🐳 Amazon ECR

The WordPress image is stored in Amazon Elastic Container Registry.

Deployment flow:

```text
Docker
   │
   ▼
WordPress Image
   │
   ▼
Amazon ECR
   │
   ▼
Kubernetes
```

The image was pushed to ECR and successfully pulled from the EC2 instance.

Example verification:

```bash
aws ecr describe-images \
  --repository-name wordpress-k8s \
  --region us-east-1
```

---

# ☸️ Kubernetes Resources

The Helm chart creates the following Kubernetes resources:

```text
WordPress
├── Deployment
└── Service

MariaDB
├── StatefulSet
├── Service
└── PersistentVolumeClaim

Application
├── Secret
└── Ingress
```

---

# ⛵ Helm

The entire application is packaged as a Helm chart.

```text
.
├── Chart.yaml
├── values.yaml
├── .helmignore
└── templates/
    ├── mariadb-service.yaml
    ├── mariadb-statefulset.yaml
    ├── secret.yaml
    ├── wordpress-deployment.yaml
    ├── wordpress-ingress.yaml
    └── wordpress-service.yaml
```

## Helm Validation

The chart was validated using:

```bash
helm lint .
```

and:

```bash
helm template wordpress .
```

Both commands complete successfully.

## Install

```bash
helm install wordpress . \
  -n wordpress \
  --create-namespace \
  --set mariadb.rootPassword='<ROOT_PASSWORD>' \
  --set mariadb.password='<PASSWORD>'
```

> Do not commit real passwords to Git.

---

# 🌐 NGINX Ingress

NGINX Ingress Controller is used to route external HTTP traffic to WordPress.

Ingress configuration:

```text
Host: wordpress.local
Port: 80
Class: nginx
```

Check the Ingress:

```bash
kubectl get ingress -n wordpress
```

Test the application:

```bash
curl -H "Host: wordpress.local" \
  http://$(minikube ip):31786/
```

A successful request redirects to:

```text
/wp-admin/install.php
```

Traffic flow:

```text
Client
  │
  ▼
NGINX Ingress
  │
  ▼
WordPress Service
  │
  ├──────────────┐
  ▼              ▼
WordPress Pod  WordPress Pod
  │              │
  └──────┬───────┘
         ▼
MariaDB Service
         │
         ▼
MariaDB Pod
```

---

# ❤️ Health Probes

WordPress includes Kubernetes health probes.

### Readiness Probe

The readiness probe determines whether a Pod is ready to receive traffic.

```text
Ready → Service can send traffic
Not Ready → Service stops routing traffic
```

### Liveness Probe

The liveness probe allows Kubernetes to detect an unhealthy container and restart it when required.

These probes improve the application's availability and resilience.

---

# 📊 Monitoring

Monitoring is implemented using `kube-prometheus-stack`.

The monitoring stack includes:

* Prometheus
* Grafana
* Alertmanager
* kube-state-metrics
* Node Exporter

Verify the monitoring stack:

```bash
kubectl get pods -n monitoring
```

---

# 📈 Grafana Dashboard

The Grafana dashboard contains application and cluster monitoring panels.

## 1. WordPress Container Uptime

PromQL:

```promql
kube_pod_container_status_ready{namespace="wordpress"}
```

The metric represents container readiness:

```text
1 = Ready
0 = Not Ready
```

The Grafana legend uses Pod names to make the panel easier to read.

---

## 2. Cluster CPU Usage

```promql
100 * (1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])))
```

This panel shows cluster CPU utilization over time.

---

## 3. Cluster Memory Usage

```promql
100 * (
  1 - (
    node_memory_MemAvailable_bytes /
    node_memory_MemTotal_bytes
  )
)
```

This panel shows cluster memory utilization over time.

---

# 🔍 Verification

## Application

```bash
kubectl get pods -n wordpress
kubectl get svc -n wordpress
kubectl get ingress -n wordpress
kubectl get pvc -n wordpress
```

Expected state:

```text
WordPress replicas:  2/2 Running
MariaDB:             1/1 Running
PVC:                 Bound
Ingress:             nginx
```

## Monitoring

```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

## Helm

```bash
helm lint .
helm template wordpress .
helm list -n wordpress
```

## ECR

```bash
aws ecr describe-images \
  --repository-name wordpress-k8s \
  --region us-east-1
```

---

# 📸 Screenshots

Project evidence and screenshots are stored in the `K8s-project/` directory.

![1](K8s/minikube.png)

```

Screenshots demonstrate:

* Helm validation
* ECR image availability
* Running Kubernetes workloads
* Persistent storage
* NGINX Ingress
* Grafana monitoring

---

# ✅ Definition of Done

| Requirement                          | Status |
| ------------------------------------ | ------ |
| Application is reachable and running | ✅      |
| WordPress Deployment                 | ✅      |
| 2 WordPress replicas                 | ✅      |
| MariaDB StatefulSet                  | ✅      |
| PersistentVolumeClaim                | ✅      |
| Kubernetes Services                  | ✅      |
| NGINX Ingress                        | ✅      |
| Amazon ECR image                     | ✅      |
| Helm deployment                      | ✅      |
| Grafana monitoring                   | ✅      |
| Container uptime monitoring          | ✅      |
| Cluster CPU monitoring               | ✅      |
| Cluster memory monitoring            | ✅      |
| Health probes                        | ✅      |
| Git repository                       | ✅      |

---

# 🧹 Cleanup

Remove the WordPress deployment:

```bash
helm uninstall wordpress -n wordpress
```

Remove the monitoring stack:

```bash
helm uninstall kube-prom-stack -n monitoring
```

Remove the application namespace:

```bash
kubectl delete namespace wordpress
```

---

# 🚀 Final Result

This project demonstrates the migration of a Docker Compose-based WordPress application into Kubernetes using AWS and cloud-native tooling.

The final solution provides:

* Declarative Kubernetes resources
* Helm-based deployment
* Stateful database persistence
* Two WordPress replicas
* Service-based networking
* NGINX ingress routing
* Amazon ECR image management
* Kubernetes health probes
* Prometheus-based monitoring
* Grafana dashboards
* Cluster-level observability

