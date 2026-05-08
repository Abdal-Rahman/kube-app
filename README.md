# Multi-Tier Application Deployment on Kubernetes

Kubernetes manifests for deploying a production-style, multi-tier Java web application stack with a relational database, caching layer, and message broker — all orchestrated as independent services within a Kubernetes cluster.

## Architecture

```
                    ┌─────────────────┐
       HTTP ──────► │    vproapp      │  (Java/Tomcat, port 8080)
                    │   Deployment    │
                    └────────┬────────┘
                             │ depends on (init containers verify DNS)
            ┌────────────────┼────────────────┐
            ▼                ▼                ▼
     ┌─────────────┐  ┌──────────────┐  ┌──────────────┐
     │   vprodb    │  │ vprocache01  │  │   RabbitMQ   │
     │    MySQL    │  │  Memcached   │  │  Message Q   │
     │  ClusterIP  │  │  ClusterIP   │  │  ClusterIP   │
     └─────────────┘  └──────────────┘  └──────────────┘
```

## Tech Stack

| Component | Technology | Purpose |
|---|---|---|
| Application | Java / Spring MVC / Tomcat | Web tier |
| Database | MySQL | Persistent data storage |
| Cache | Memcached | Session & query caching |
| Message Broker | RabbitMQ | Async task processing |
| Orchestration | Kubernetes | Container management |
| Secrets | Kubernetes Secrets | Credential management |

## Repository Structure

```
├── vproappdep.yaml       # App deployment (with init containers)
├── vproapp-service.yaml  # App service (LoadBalancer/NodePort)
├── vprodbdep.yaml        # MySQL deployment
├── db-CIP.yaml           # MySQL ClusterIP service
├── mcdep.yaml            # Memcached deployment
├── mc-CIP.yaml           # Memcached ClusterIP service
├── rmq-dep.yaml          # RabbitMQ deployment
├── rmq-CIP.yaml          # RabbitMQ ClusterIP service
└── app-secret.yaml       # Kubernetes Secret (DB credentials)
```

## Kubernetes Concepts Demonstrated

- **Deployments** for each service tier with replica management
- **ClusterIP services** for internal service-to-service communication
- **Init containers** on the app pod to enforce startup order — the app waits until `vprodb` and `vprocache01` DNS names resolve before starting
- **Kubernetes Secrets** for managing database credentials without hardcoding
- **Multi-tier separation of concerns** — each component is independently scalable and replaceable

## Init Container Pattern

The application deployment uses init containers to handle service dependencies gracefully:

```yaml
initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup vprodb; do echo waiting for mydb; sleep 2; done"]
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup vprocache01; do echo waiting for mydb; sleep 2; done"]
```

This ensures the app pod only starts after both the database and cache services are reachable, preventing crash loops on cold starts.

## How to Deploy

### Prerequisites
- A running Kubernetes cluster (Minikube, KOPS, EKS, GKE, or AKS)
- `kubectl` configured to point to your cluster

### Steps

```bash
# 1. Create secrets (update values with your base64-encoded credentials first)
kubectl apply -f app-secret.yaml

# 2. Deploy backend services
kubectl apply -f vprodbdep.yaml
kubectl apply -f db-CIP.yaml
kubectl apply -f mcdep.yaml
kubectl apply -f mc-CIP.yaml
kubectl apply -f rmq-dep.yaml
kubectl apply -f rmq-CIP.yaml

# 3. Deploy the application
kubectl apply -f vproappdep.yaml
kubectl apply -f vproapp-service.yaml

# 4. Verify all pods are running
kubectl get pods
kubectl get services
```

### Accessing the Application
Once all pods are in `Running` state, access the app via the `vproapp-service` external IP or NodePort.

## Key Concepts Demonstrated

- **Microservice decomposition**: each tier runs as an independent, replaceable Kubernetes workload
- **Declarative infrastructure**: entire stack defined as code, reproducible in any cluster
- **Resilient startup ordering**: init containers prevent premature application launch
- **Secure credential handling**: secrets kept out of deployment manifests
