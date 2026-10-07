Most people explain GitOps like this:  "GitOps is an operational framework that uses Git as the single source of truth..."
That tells you absolutely nothing.
# Imagine you are the IT guy in your company

Your job is to deploy a website.

```
portfolio/
 ├── index.html
 ├── css/
 └── js/
```

You have an AWS EC2 server. Normally you do something like

```
git pull
docker build
docker run
```

or

```
scp files
ssh server
restart nginx
```

You're logging into servers. Changing files manually. Restarting services manually. If another IT guy changes something... Nobody knows what happened. This is **traditional deployment.**
# Problem 1

Suppose you accidentally delete nginx. `rm -rf /etc/nginx`
Website dies. Git still contains everything. But Git doesn't magically fix your server. You must SSH and repair it.
# Problem 2

Another admin installs Apache. `sudo apt install apache2` Now your server looks like

```
Git says

Nginx
Docker
Version 1.0

Reality

Apache
Docker
Version 0.8
```

Git and reality are different. This is called **configuration drift.** Nobody notices until something breaks.
# Problem 3

Developer updates code. `git push` Now someone must remember

```
SSH
git pull
restart service
```

If they forget...  Production stays old. This is where GitOps comes in.
# GitOps says

**Stop logging into servers.** Instead. Tell the server "Whatever is inside Git. Make yourself look exactly like that." That's it. That is GitOps.
Think of Git as the blueprint.

```
Git

Docker image
Deployment
Config
Secrets
Ingress
↓
Server reads Git
↓
Server changes itself
```

Notice something important. YOU are no longer pushing changes. The SERVER is pulling changes. Huge difference.
Traditional

```
Developer
↓
SSH
↓
Server
```

GitOps

```
Developer
↓
Git
↓
Server watches Git
```

Nobody SSHs.

# But how does the server know Git changed?

Exactly. Some software constantly checks Git. That software is called **ArgoCD**.
# What is ArgoCD?

ArgoCD is basically `A robot` Seriously. Imagine this robot lives inside Kubernetes. Every few seconds it asks

```
Has Git changed?
No?
Sleep.
Has Git changed?
No?
Sleep.
Has Git changed?
YES!
Deploy.
```

That's literally its job. Without ArgoCD

```
Developer
↓
Git
↓
Human
↓
kubectl apply
```

With ArgoCD

```
Developer
↓
Git
↓
ArgoCD
↓
Cluster
```

No humans.
# Let's see a real example

Suppose Git contains

```
deployment.yaml
replicas: 2
```

Current cluster `2 Pods` Everything matches.
Developer changes

```
replicas: 5
```

Pushes to Git. Nothing else. No kubectl. No SSH. No Jenkins deployment. Nothing. ArgoCD notices

```
Git
5 Pods
Cluster
2 Pods
```

Difference detected. So ArgoCD runs `kubectl apply` internally. Cluster becomes `5 Pods` , Done.
# Another example

Git

```
image:
nginx:1.30
```

Cluster `nginx:1.29`
ArgoCD notices

```
Desired
1.30
Actual
1.29
```

Updates automatically.
# What if someone manually changes the cluster?

Suppose your teammate runs `kubectl scale deployment app --replicas=20`
Cluster `20 Pods`
Git `5 Pods`
ArgoCD immediately says Nope.
Changes cluster back to `5 Pods` Because Git is the boss.
Humans are not.
Think of Git as `Desired State`
Cluster is `Actual State`
ArgoCD continuously does

```
Desired == Actual ?
YES
Do nothing.
NO
Fix it.
```

That is literally ArgoCD's whole purpose.
# Where does Kubernetes come in?

ArgoCD doesn't deploy Docker containers directly.
It deploys Kubernetes objects.
Git contains things like

```
Deployment
Service
Ingress
ConfigMap
Secret
Namespace
```

Example

```
apiVersion: apps/v1
kind: Deployment
metadata:
name: portfolio
```

ArgoCD reads it.
Runs `kubectl apply`
Cluster updates.
# Complete flow

Developer changes code.

```
app.py
↓
Pushes to GitHub.
↓
GitHub Actions builds Docker image.
↓
Pushes image to Docker Hub (or Amazon ECR).
↓
Updates Kubernetes YAML.
image:
portfolio:v2
↓
Commits YAML.
↓
Git changes.
↓
ArgoCD detects change.
↓
Applies YAML.
↓
Kubernetes pulls new image.
↓
Pods restart.
↓
Website updated.
```

Notice that ArgoCD is watching the **deployment manifests**, not your application source code.

# Why do companies love GitOps?

Imagine your cluster crashes completely. No problem.
Create a new Kubernetes cluster. Install ArgoCD. Point it at Git.
ArgoCD recreates everything:

- Deployments
- Services
- Ingresses
- ConfigMaps
- Namespaces
- Autoscalers
- RBAC (if stored in Git)

No manual reconstruction because Git already describes the desired state.

# Where does Terraform fit?

This is where many beginners get confused. Terraform creates **infrastructure**.

```
AWS

VPC
Subnets
Security Groups
EKS Cluster
Load Balancer
```

ArgoCD deploys **applications** onto that infrastructure.

```
Kubernetes

Deployment
Service
Ingress
ConfigMap
Secret
```

So a common DevOps flow is:

```
Terraform
    │
    ▼
Creates AWS Infrastructure
    │
    ▼
Creates Kubernetes Cluster (EKS)
    │
    ▼
Install ArgoCD
    │
    ▼
ArgoCD watches Git
    │
    ▼
Deploys applications
```

# Since you're building your AWS portfolio, this is the architecture you'll eventually have

```
                  Developer
                      │
                  git push
                      │
                      ▼
                 GitHub Repository
          (Source + Kubernetes Manifests)
                      │
        GitHub Actions builds Docker image
                      │
                      ▼
                  Amazon ECR
                      │
      Updates image tag in Kubernetes YAML
                      │
                      ▼
                GitHub Repository
                      │
          ArgoCD continuously watches Git
                      │
                      ▼
              Amazon EKS Cluster
                      │
                  Kubernetes
                      │
                  Running Pods
                      │
                 Application Live
```

This separation is important:

|Tool|Responsibility|
|---|---|
|Git|Stores the desired application configuration|
|GitHub Actions/Jenkins|Builds and publishes Docker images|
|Terraform|Creates and manages AWS infrastructure|
|Kubernetes|Runs containers|
|ArgoCD|Continuously makes Kubernetes match what's in Git|
