# What is an Architecture?

Most people think architecture means: `AWS diagram with EC2 → ALB → RDS`
No. Architecture means **How everything works together from the moment a developer writes code until customers use the application.**
Architecture answers questions like:

- Where is code stored?
- How is it built?
- How is infrastructure created?
- How is application deployed?
- How are secrets stored?
- How is traffic routed?
- How are failures handled?
- How is monitoring done?
- How are backups taken?
- How is security enforced?

A DevOps engineer's job is designing and maintaining this entire pipeline.
# Types of Architectures

There isn't just one architecture.

```
1. Monolithic Architecture
2. Three-Tier Architecture
3. Microservices Architecture
4. Kubernetes Architecture
5. Serverless Architecture
6. Event-Driven Architecture
7. Hybrid Architecture
8. GitOps Architecture
9. Multi-cloud Architecture
10. Enterprise Platform Architecture
```

# 1. Monolithic Architecture

This is where almost everyone starts. Imagine a shopping website. Everything lives in one project.

```
shopping-app/
login/
cart/
payment/
orders/
admin/
inventory/
search/
notifications/
```

Everything builds together. One deployment. One application. One server (or multiple identical servers).
Example

```
Customer
↓
Load Balancer
↓
EC2
↓
Application
↓
MySQL
```

```
             Internet
                 │
           Load Balancer
                 │
         ----------------
         │              │
       EC2            EC2
         │              │
         -------MySQL------
```

Pipeline

```
Developer
↓
GitHub
↓
CI Pipeline
↓
Build entire application
↓
Create Docker Image
↓
Deploy complete application
```

Advantages

```
Simple
Easy debugging
Easy deployment
Cheap
```

Disadvantages

```
Huge deployments
Slow builds
One bug can crash everything
Scaling is difficult
```

Example

```
Amazon in 1998
Facebook initially
Many startups
```

# 2. Three Tier Architecture

One of the most common architectures. Application is divided into three parts.

```
Presentation Layer
Business Layer
Database Layer
```

Diagram

```
Users
↓
Frontend
↓
Backend API
↓
Database
```

More realistic

```
Users
↓
CloudFront
↓
React
↓
ALB
↓
Node API
↓
RDS
```

AWS

```
CloudFront
↓
S3
↓
ALB
↓
EC2/ECS/EKS
↓
RDS
```

Responsibilities

Presentation

```
React
Angular
Vue
```

Business Layer

```
Java
Spring
.NET
Python
Node
Go
```

Database

```
MySQL
Postgres
MongoDB
```

# 3. Microservices Architecture

This is where most companies are today. Instead of `One application` , You create

```
20
50
100
300 services
```

Example

Amazon

```
Login Service
Order Service
Inventory Service
Payment Service
Shipping Service
Notification Service
Product Service
Recommendation Service
```

Each is its own application. Own repository. Own deployment. Own CI/CD. Own Docker image. Sometimes own database.
Diagram

```
Internet
↓
API Gateway
↓
-------------------------------
Login
Orders
Products
Inventory
Payments
Shipping
Notification
↓
Each service owns database
```

Advantages

```
Independent deployments
Independent scaling
Independent teams
Technology freedom
Fault isolation
```

Disadvantages

```
Much harder
Need Kubernetes
Need service discovery
Need logging
Need tracing
Need monitoring
Need message queues
Need API gateway
```

This is where DevOps engineers become valuable.

# Microservice Deployment

Developer modifies `payment-service`
Pipeline

```
GitHub
↓
GitHub Actions
↓
Docker Build
↓
Push to Registry
↓
Deploy only payment-service
↓
Kubernetes Rolling Update
```

Notice , Other services remain untouched.

# 4. Kubernetes Architecture

This isn't an application architecture. It's an infrastructure architecture.

Example

```
Developer
↓
GitHub
↓
CI
↓
Docker Image
↓
Registry
↓
Kubernetes Cluster
↓
Pods
↓
Users
```

Inside Kubernetes

```
Ingress
↓
Service
↓
Deployment
↓
ReplicaSet
↓
Pods
↓
Containers
```

Worker Nodes

```
Node1
Pod A
Pod B
Pod C
------------
Node2
Pod D
Pod E
Pod F
------------
Node3
Pod G
Pod H
Pod I
```

Supporting Components

```
Ingress Controller
Service Mesh
Metrics Server
Prometheus
Grafana
Loki
Fluentd
ArgoCD
Cert Manager
External DNS
```

This becomes Platform Engineering.

# 5. Serverless Architecture

No servers to manage.
Example

```
Customer uploads image
↓
API Gateway
↓
Lambda
↓
S3
↓
DynamoDB
```

Event flow

```
Image uploaded
↓
S3 Event
↓
Lambda
↓
Resize Image
↓
Store
↓
Notify User
```

DevOps here focuses on Infrastructure as Code, IAM, Monitoring, CI/CD Not servers.

# 6. Event Driven Architecture

Instead of

```
Service A
↓
calls
↓
Service B
```

Everything communicates through events.

Example
Customer places order.

Instead of

```
Order
↓
Payment
↓
Shipping
↓
Inventory
↓
Notification
```

It becomes

```
Order Created Event
↓
Kafka
↓
Inventory listens
↓
Payment listens
↓
Shipping listens
↓
Email listens
```

Nobody calls each other directly.

Advantages

```
Loose coupling
Easy scaling
Fault tolerance
```


Tools

```
Kafka
RabbitMQ
SQS
SNS
NATS
```

# 7. GitOps Architecture

Very popular today. Old way , Pipeline deploys directly.

```
GitHub Actions
↓
kubectl apply
```

GitOps

```
GitHub
↓
Manifest Repository
↓
ArgoCD
↓
Kubernetes
```

```
Developer
↓
Change YAML
↓
Git Push
↓
ArgoCD notices
↓
Deploys automatically
```

Everything is driven by Git. Git becomes the source of truth.

# 8. Hybrid Architecture

Some applications `On-premise`, Some `AWS` , Some `Azure`
Diagram

```
Office
↓
VPN
↓
AWS
↓
Azure
↓
On-prem Datacenter
```

Large companies
Banks
Insurance
Government
Almost always use hybrid.

# 9. Multi-cloud Architecture

Application exists in AWS, Azure, Google Cloud
Example

```
Traffic
↓
Route53
↓
AWS
↓
Azure
↓
GCP
```

Reasons

```
Disaster Recovery
Compliance
Avoid vendor lock-in
Latency
```

# 10. Enterprise Platform Architecture

This is what companies like Microsoft, Google, Netflix, Uber, Amazon build. Developers don't manage infrastructure. Instead, Platform Team provides

```
GitHub
↓
CI
↓
Docker Registry
↓
Kubernetes
↓
Secrets
↓
Monitoring
↓
Logging
↓
Alerting
↓
Security
↓
Service Mesh
↓
DNS
↓
Ingress
↓
Templates
```

Developers simply `git push` , Everything else happens automatically.

# DevOps End-to-End Architecture

This is what many modern companies use:

```
                    Developers
                         │
                         ▼
                     GitHub/GitLab
                         │
               Pull Request / Review
                         │
                         ▼
                  CI Pipeline
      (GitHub Actions/Jenkins/GitLab CI)
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
     Linting         Unit Tests       Security Scan
                                         │
                                         ▼
                                  Build Docker Image
                                         │
                                         ▼
                                 Container Registry
                              (Docker Hub/ECR/GHCR)
                                         │
                                         ▼
                           Infrastructure (Terraform)
                                         │
                                         ▼
                         Kubernetes / ECS / VMs Created
                                         │
                                         ▼
                        CD (Argo CD / Flux / Spinnaker)
                                         │
                                         ▼
                            Kubernetes Cluster
                         ┌───────────┬───────────┐
                         ▼           ▼           ▼
                    Namespace    Namespace   Namespace
                    (Dev)         (Stage)      (Prod)
                         │           │           │
                         ▼           ▼           ▼
                    Deployments  Services   Ingress
                         │
                         ▼
                        Pods
                         │
                         ▼
                   Application Containers
                         │
         ┌───────────────┼────────────────┐
         ▼               ▼                ▼
      PostgreSQL       Redis          Message Queue
                                        (Kafka/SQS)
                         │
                         ▼
                  Monitoring Stack
       Prometheus → Grafana → Alertmanager
                         │
                         ▼
                  Logging Stack
          Fluent Bit → Loki/Elasticsearch
                         │
                         ▼
                    Notifications
              Slack / Teams / Email / PagerDuty
```

---

# How Everything Connects

Let's walk through the lifecycle of a single code change.

### 1. A developer writes code

They create a new feature branch from `main`, make changes, commit them, and push the branch to the remote repository.

### 2. A Pull Request is opened

Another developer reviews the code. Automated checks (linting, unit tests, security scans) run in the CI pipeline. If any required check fails, the PR cannot be merged.

### 3. Code is merged

Once approved, the CI pipeline starts again from the `main` branch.

Typical CI tasks:

- Install dependencies
- Run tests
- Run static code analysis
- Scan for vulnerabilities
- Build the application
- Build a Docker image
- Tag the image (for example with the Git commit SHA)
- Push the image to a container registry

### 4. Infrastructure is prepared

If infrastructure changes are needed, Terraform plans and applies changes to cloud resources such as VPCs, subnets, load balancers, databases, IAM roles, and Kubernetes clusters.

### 5. Continuous Delivery

In a GitOps setup, the deployment manifests are updated to reference the new image tag. A deployment controller such as Argo CD detects the Git change and reconciles the cluster state, performing a rolling update.

### 6. Traffic reaches the application

A user request typically flows like this:

```
Browser
  ↓
DNS
  ↓
CDN (optional)
  ↓
Load Balancer
  ↓
Ingress Controller
  ↓
Kubernetes Service
  ↓
Pod
  ↓
Application
  ↓
Database / Cache / Message Queue
```

### 7. Observability

Throughout this flow:

- **Metrics** are scraped by Prometheus and visualized in Grafana.
- **Logs** are collected by agents such as Fluent Bit and stored in Loki or Elasticsearch.
- **Distributed traces** (using technologies like OpenTelemetry) let engineers follow a request across multiple services.
- **Alerts** are triggered when thresholds or error conditions are met and sent to tools like Slack, Microsoft Teams, or PagerDuty.
