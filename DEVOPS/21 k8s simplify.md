# 👾👾👾👾👾👾👾👾👾👾👾👾👾👾👾👾👾  2.1

That paragraph is written in **Kubernetes documentation/architecture-jargon**, and it makes the topic sound much more complicated than it is.

The key idea is simply:

> **Some Kubernetes components can all work at the same time. Others have multiple copies for high availability, but only one copy is allowed to actively make decisions at a time.**

Let's break it down from zero.

---

# 1. First: Why do we have multiple copies?

Imagine your Kubernetes control plane has only one API server:

```text
             Kubernetes
                 │
                 ▼
            API Server
                 │
                 ▼
               etcd
```

If that API server crashes:

```text
             Kubernetes
                 │
                 X
            API Server ❌
```

Your applications might still be running, but you can't properly manage the cluster.

So production Kubernetes clusters commonly have multiple control-plane instances:

```text
              Load Balancer
              /     |     \
             /      |      \
            ▼       ▼       ▼
         API-1    API-2    API-3
```

This is called **High Availability (HA)**.

---

# 2. The important distinction: Active-Active vs Active-Passive

Forget Kubernetes for a moment.

Imagine you have three employees handling support tickets.

### Active-Active

All three can handle tickets simultaneously:

```text
Ticket
  │
  ├──► Employee A
  ├──► Employee B
  └──► Employee C
```

Everyone is working.

### Active-Passive

Only one person is allowed to make decisions:

```text
          Active
         Employee A
             │
             │
             X  ← fails
             
          Passive
         Employee B
             │
             ▼
          becomes
           Active
```

Employee B was waiting in standby.

That's basically the distinction.

---

# 3. kube-apiserver = Active-Active

This one is relatively easy.

Suppose you have:

```text
                    Load Balancer
                   /      |      \
                  ▼       ▼       ▼
               API-1    API-2    API-3
```

All three API servers are active.

A request can go to any of them.

For example:

```bash
kubectl get pods
```

could go:

```text
kubectl
   │
   ▼
Load Balancer
   │
   ▼
API-2
```

Another request:

```bash
kubectl get nodes
```

could go:

```text
kubectl
   │
   ▼
Load Balancer
   │
   ▼
API-1
```

And another:

```text
kubectl apply -f app.yaml
```

could go to:

```text
API-3
```

All three can process requests.

---

# 4. Why can API servers do this?

Because the API server itself doesn't need to remember your session/request history in local memory.

That's what **stateless** means in this context.

Think:

```text
API-1
API-2
API-3
```

Any one can receive a request.

The cluster's persistent state is stored in:

```text
API Server
     │
     ▼
   etcd
```

So if API-2 dies:

```text
Load Balancer
   │
   ├── API-1 ✓
   ├── API-2 ❌
   └── API-3 ✓
```

Requests simply go to API-1 or API-3.

That's why multiple API servers can be active simultaneously.

---

# 5. Now etcd is different

This is where that paragraph starts getting confusing.

etcd is a **distributed database**.

Suppose you have:

```text
etcd-1
etcd-2
etcd-3
```

They work together as a cluster.

```text
             etcd cluster

          ┌─────────────┐
          │   etcd-1    │
          │   LEADER    │
          └──────┬──────┘
                 │
          ┌──────┴──────┐
          ▼             ▼
     etcd-2          etcd-3
    follower         follower
```

But here's the important part:

> **They are all running, but they don't all independently make write decisions.**

One node becomes the **Raft leader**.

---

# 6. What is Raft?

Don't worry about the mathematical details.

For Kubernetes purposes, think:

> **Raft is a consensus mechanism that lets multiple etcd nodes agree on the cluster's state.**

For example:

```text
etcd-1 = Leader
etcd-2 = Follower
etcd-3 = Follower
```

Suppose Kubernetes needs to write something to etcd.

The write goes through the leader:

```text
              Write
                │
                ▼
           etcd-1
           LEADER
            /   \
           ▼     ▼
       etcd-2   etcd-3
       follower follower
```

The leader replicates the information to the followers.

---

# 7. What happens if etcd leader dies?

Suppose:

```text
etcd-1 ❌
```

The remaining nodes can elect another leader:

```text
Before:

etcd-1 = Leader
etcd-2 = Follower
etcd-3 = Follower


After:

etcd-1 ❌

etcd-2 = Leader
etcd-3 = Follower
```

That's how etcd provides fault tolerance.

---

# 8. Why did your document call etcd "Active-Active"?

This wording is potentially confusing.

You can think of all etcd members as **participating members of the cluster**, but Raft still has a **single leader for a given term**, and writes are coordinated through that leader.

So:

```text
etcd
├── Leader
├── Follower
└── Follower
```

is much more useful for learning than simply memorizing:

> "etcd = Active-Active."

The important thing to remember is:

**etcd has multiple members, but Raft elects one leader.**

---

# 9. kube-scheduler = Active-Passive

Now we get to the part that sounds scary.

Suppose you have:

```text
Scheduler-1
Scheduler-2
Scheduler-3
```

All three processes can be running.

But Kubernetes doesn't want all three independently scheduling the same Pod.

Imagine:

```text
Pod needs a node.

Scheduler-1 → Node 1
Scheduler-2 → Node 2
Scheduler-3 → Node 3
```

That could cause conflicting decisions.

So Kubernetes uses **leader election**.

For example:

```text
Scheduler-1 = LEADER ✓
Scheduler-2 = standby
Scheduler-3 = standby
```

Only Scheduler-1 performs the active scheduling work.

---

# 10. What happens if Scheduler-1 dies?

Before:

```text
Scheduler-1 = ACTIVE ✓
Scheduler-2 = STANDBY
Scheduler-3 = STANDBY
```

Scheduler-1 crashes:

```text
Scheduler-1 ❌
Scheduler-2 = STANDBY
Scheduler-3 = STANDBY
```

The remaining schedulers perform leader election.

Suppose Scheduler-2 wins:

```text
Scheduler-1 ❌
Scheduler-2 = ACTIVE ✓
Scheduler-3 = STANDBY
```

Now Scheduler-2 takes over.

That's what **Active-Passive** means here.

---

# 11. kube-controller-manager works similarly

Suppose you have:

```text
Controller Manager 1
Controller Manager 2
Controller Manager 3
```

Only one is the active leader:

```text
Controller-1 = ACTIVE ✓
Controller-2 = STANDBY
Controller-3 = STANDBY
```

Why?

Because controllers are constantly making changes.

For example, a Deployment says:

```text
Desired Pods = 3
```

Actual:

```text
Pod 1 ✓
Pod 2 ✓
Pod 3 ❌
```

The controller notices:

```text
Desired = 3
Actual  = 2
```

and takes action to create another Pod.

You don't want three independent controller-manager instances all trying to reconcile the same thing simultaneously.

So:

```text
One active controller manager
        +
Standby controller managers
```

---

# 12. Where does the leader election information live?

Your paragraph says:

> "leader election via etcd leases"

This basically means the scheduler/controller-manager uses a Kubernetes coordination mechanism to determine:

> "Who is currently the leader?"

Conceptually:

```text
Scheduler-1
Scheduler-2
Scheduler-3
     │
     ▼
Leader Election
     │
     ▼
Scheduler-1 = Leader
```

A lease is essentially a time-limited claim to leadership.

If the leader stops renewing its lease:

```text
Scheduler-1
     │
     X
doesn't renew
```

another instance can take over.

---

# 13. Put everything together

Now your original paragraph becomes much easier:

```text
                 External Load Balancer
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
           API-1       API-2      API-3
             ✓           ✓          ✓
          ACTIVE       ACTIVE     ACTIVE
          
             └──────────┬──────────┘
                        │
                        ▼
                  etcd cluster
                 ┌──────┼──────┐
                 ▼      ▼      ▼
              etcd-1  etcd-2  etcd-3
              Leader  Follower Follower


        Scheduler cluster
        ┌────────┬────────┐
        ▼        ▼        ▼
      Sched-1  Sched-2  Sched-3
       LEADER   Standby   Standby


   Controller Manager cluster
        ┌────────┬────────┐
        ▼        ▼        ▼
      CM-1     CM-2     CM-3
      LEADER   Standby  Standby
```

---

# 14. The easiest way to remember it

|Component|Multiple instances?|Who works?|
|---|---|---|
|**kube-apiserver**|Yes|**All can work simultaneously**|
|**etcd**|Yes|One Raft leader coordinates writes; followers replicate|
|**kube-scheduler**|Yes|**One active leader**|
|**kube-controller-manager**|Yes|**One active leader**|

So the simplified mental model is:

```text
API Server
   ↓
"Everyone can serve requests."

etcd
   ↓
"Multiple members agree using Raft;
 one is the current leader."

Scheduler
   ↓
"Multiple copies exist;
 one is active."

Controller Manager
   ↓
"Multiple copies exist;
 one is active."
```

---

# 15. Why Kubernetes uses two different approaches

This is the actual architectural reason.

### API Server

Requests are independent enough that multiple instances can process them:

```text
Request A → API-1
Request B → API-2
Request C → API-3
```

So **Active-Active** makes sense.

### Scheduler

Scheduling involves making decisions about where Pods should run.

You don't want several independent schedulers making conflicting decisions.

So:

```text
One leader
Others standby
```

### Controller Manager

Controllers actively modify cluster resources.

Again, you generally want one leader executing those control loops.

So:

```text
One leader
Others standby
```

### etcd

etcd has its own distributed-consensus architecture:

```text
             etcd
              │
       ┌──────┼──────┐
       ▼      ▼      ▼
    Leader  Follower Follower
```

The Raft protocol makes the cluster agree on state and elect a leader.

---

## The one diagram I'd memorize

If you're learning Kubernetes, forget the complicated paragraph for a moment and memorize this:

```text
                         LOAD BALANCER
                              │
                ┌─────────────┼─────────────┐
                ▼             ▼             ▼
             API-1          API-2          API-3
             ACTIVE         ACTIVE         ACTIVE
                │             │             │
                └─────────────┼─────────────┘
                              │
                              ▼
                         etcd cluster
                       ┌──────┼──────┐
                       ▼      ▼      ▼
                    Leader  Follower Follower
                             
                             
              Scheduler Cluster
                 ┌────┬────┐
                 ▼    ▼    ▼
               S-1   S-2   S-3
              LEADER standby standby


         Controller Manager Cluster
                 ┌────┬────┐
                 ▼    ▼    ▼
               C-1   C-2   C-3
              LEADER standby standby
```

**The key distinction is not really "active vs passive" everywhere.**

It's:

> **API servers: multiple instances actively serve requests.**  
> **etcd: multiple members form a consensus cluster with one Raft leader.**  
> **Scheduler/controller-manager: multiple instances may exist, but leader election ensures one active instance at a time.**


# 👾👾👾👾👾👾👾👾👾👾👾👾👾 2.2

Yes. The problem is that this document is throwing **15 Kubernetes concepts at you at once**. The actual idea behind this section is much simpler.

Let's forget the jargon and understand what **kube-apiserver actually does when you run a command**.

---

# 1. First: What is kube-apiserver?

Think of the API Server as the **front door of the Kubernetes cluster**.

```text
                 Kubernetes Cluster
                        │
                        │
                 ┌──────▼──────┐
                 │ kube-apiserver│
                 │  "Front Door" │
                 └──────┬───────┘
                        │
             ┌──────────┼──────────┐
             ▼          ▼          ▼
            etcd    Scheduler   Controllers
```

You don't normally walk directly into etcd, Scheduler, or Controller Manager.

You communicate with Kubernetes through the API Server.

For example:

```bash
kubectl get pods
```

or:

```bash
kubectl create deployment nginx --image=nginx
```

or:

```bash
kubectl delete pod mypod
```

All of these ultimately communicate with the **kube-apiserver**.

---

# 2. What does `kubectl` actually do?

When you type:

```bash
kubectl get pods
```

`kubectl` isn't itself looking inside the Kubernetes cluster.

Instead:

```text
You
 │
 │ kubectl get pods
 ▼
kube-apiserver
 │
 ▼
Kubernetes
```

The API Server receives an HTTP request.

Conceptually:

```text
GET /api/v1/namespaces/default/pods
```

So Kubernetes is fundamentally an **API-driven system**.

---

# 3. Now let's understand the 5 stages

Your document says:

```text
Request
   │
   ▼
1. Authentication
   │
   ▼
2. Authorization
   │
   ▼
3. Admission
   │
   ▼
4. Validation
   │
   ▼
5. Persistence
```

This is basically asking five questions:

```text
1. WHO are you?

2. ARE YOU ALLOWED to do this?

3. SHOULD Kubernetes MODIFY or REJECT the request?

4. IS the object structurally valid?

5. SAVE it.
```

That's the whole concept.

---

# 4. Stage 1 — Authentication

Authentication means:

> **"Who are you?"**

Suppose you run:

```bash
kubectl get pods
```

The API Server needs to know who is making the request.

For example:

```text
Request
   │
   ▼
API Server
   │
   ▼
Authentication
   │
   ▼
"Koushik"
```

Now Kubernetes knows the identity of the caller.

Authentication can use different mechanisms.

Your document lists:

- X.509 certificates
    
- ServiceAccount tokens
    
- OIDC
    
- Webhook token authentication
    

You **do not need to learn all of these immediately**.

For now remember:

> **Authentication = WHO are you?**

---

# 5. Authentication vs Authorization

This distinction is extremely important.

Imagine you enter an office.

Security asks:

> "Who are you?"

You show your employee ID.

That's **authentication**.

Then you try to enter the server room.

Security asks:

> "Are you allowed to enter the server room?"

That's **authorization**.

Same thing in Kubernetes.

```text
Authentication
      ↓
Who are you?

Authorization
      ↓
What are you allowed to do?
```

---

# 6. Stage 2 — Authorization

Now Kubernetes knows:

```text
User = Koushik
```

It asks:

> **"Can Koushik perform this operation?"**

Suppose you run:

```bash
kubectl get pods
```

Kubernetes checks whether you have permission to:

```text
verb: get
resource: pods
```

For example:

```text
User: Koushik

Can:
✓ get pods
✓ list pods
✓ get deployments

Cannot:
✗ delete namespaces
✗ modify cluster-wide configuration
```

This is where **RBAC** comes into play.

RBAC = **Role-Based Access Control**.

So:

```text
Authentication
    ↓
WHO are you?

Authorization / RBAC
    ↓
WHAT are you allowed to do?
```

---

# 7. Stage 3 — Admission Control

This is the part that confuses most beginners.

Suppose you're allowed to create a Pod.

Authorization says:

```text
✓ You are allowed to create Pods.
```

But Kubernetes can still say:

> "Hold on. Before I accept this Pod, I need to check or modify it."

That's **Admission Control**.

```text
Request
   │
   ▼
Authentication ✓
   │
   ▼
Authorization ✓
   │
   ▼
Admission Control
```

Admission controllers can:

- Modify the request
    
- Add defaults
    
- Check policies
    
- Reject the request
    

---

# 8. Mutating vs Validating

This is much easier than the document makes it sound.

There are basically two types.

## Mutating

> **"I'll change something."**

Example:

You submit:

```yaml
resources:
  requests: {}
```

A mutating admission controller might add default values:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
```

So:

```text
You submit object
       │
       ▼
Mutating Admission
       │
       ▼
Object modified
```

---

# 9. Validating

Validating means:

> **"I'll check it, but I won't modify it."**

For example:

```text
You request:
CPU = 100 CPUs
```

But your namespace has:

```text
CPU quota = 10 CPUs
```

The request can be rejected.

```text
Request
   │
   ▼
Validating Admission
   │
   ▼
❌ REJECT
```

So remember:

```text
Mutating
    ↓
Can CHANGE the object

Validating
    ↓
Can ACCEPT or REJECT the object
```

---

# 10. Example: ResourceQuota

Suppose your namespace has:

```text
Maximum CPU = 4 cores
```

You try to create:

```yaml
resources:
  requests:
    cpu: 8
```

Admission control can say:

```text
❌ Rejected

Requested CPU exceeds namespace quota.
```

This happens **before the object is persisted**.

---

# 11. Example: Pod Security

Suppose your organization says:

> Pods must follow a particular security policy.

You try to create an unsafe Pod.

The admission layer can reject it:

```text
Pod
 │
 ▼
Pod Security Admission
 │
 ▼
❌ Rejected
```

Again:

> Admission control is basically **the security/policy checkpoint before Kubernetes accepts the object**.

---

# 12. Stage 4 — Schema Validation

Now Kubernetes asks:

> **"Does this object have the correct structure and data types?"**

Suppose you write:

```yaml
replicas: three
```

But Kubernetes expects:

```yaml
replicas: 3
```

Because replicas is an integer.

So validation can reject the object.

```text
replicas: three
       │
       ▼
Schema validation
       │
       ▼
❌ Invalid
```

Correct:

```yaml
replicas: 3
```

---

# 13. Don't get too obsessed with the exact ordering

Your document presents it as:

```text
Authentication
↓
Authorization
↓
Admission
↓
OpenAPI Validation
↓
Persistence
```

That's useful as a learning model, but **real kube-apiserver request processing has more internal stages and details than this simplified five-box diagram**.

For your current understanding, the important concept is:

```text
Identity
   ↓
Permission
   ↓
Policy/admission
   ↓
Validation
   ↓
Store
```

---

# 14. Stage 5 — Persistence

Finally Kubernetes says:

> "Everything looks good. Save it."

Where?

**etcd.**

Remember from your previous question:

```text
kube-apiserver
       │
       ▼
      etcd
```

etcd stores Kubernetes cluster state.

For example:

```text
Deployment
replicas: 3
image: nginx
```

gets persisted as part of the cluster's state.

---

# 15. Now let's follow a REAL example

Suppose you have this:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
```

And run:

```bash
kubectl apply -f nginx.yaml
```

Let's see what happens.

---

## Step 1 — kubectl

```text
kubectl
   │
   │ HTTP request
   ▼
API Server
```

---

## Step 2 — Authentication

API Server asks:

```text
Who is making this request?
```

For example:

```text
User = Koushik
```

```text
Authentication ✓
```

---

## Step 3 — Authorization

API Server asks:

```text
Can Koushik create/update Deployments?
```

RBAC checks permissions.

```text
Authorization ✓
```

---

## Step 4 — Admission

Kubernetes checks:

```text
Does this request violate any policies?
```

Potentially:

```text
ResourceQuota
Pod Security
Mutating policies
Validating policies
etc.
```

```text
Admission ✓
```

---

## Step 5 — Validation

Kubernetes checks whether the Deployment object is structurally valid.

```text
replicas = 3 ✓
apiVersion = apps/v1 ✓
kind = Deployment ✓
```

```text
Validation ✓
```

---

## Step 6 — Store it

API Server writes the state to:

```text
etcd
```

```text
API Server
     │
     ▼
   etcd
```

Now Kubernetes knows:

```text
Desired state:

Deployment nginx
replicas = 3
```

---

# 16. But wait — are the Pods running now?

Not necessarily.

This is another extremely important distinction.

The API Server **doesn't itself create and run the containers**.

It has essentially recorded:

> "The user wants this Deployment."

Then other Kubernetes components react.

Conceptually:

```text
kubectl
   │
   ▼
API Server
   │
   ▼
etcd
   │
   ▼
Controllers notice desired state
   │
   ▼
ReplicaSet
   │
   ▼
Pods
   │
   ▼
Scheduler
   │
   ▼
Worker Node
   │
   ▼
kubelet
   │
   ▼
Container Runtime
   │
   ▼
Container
```

This distinction is **very important**.

---

# 17. Now the authentication methods

Your document suddenly starts talking about:

> X.509  
> ServiceAccount JWT  
> OIDC  
> Webhook Token Authentication

Don't let that scare you.

They are simply **different ways of proving identity**.

---

## X.509

Certificate-based identity.

Conceptually:

```text
Client
 │
 │ Certificate
 ▼
API Server
 │
 ▼
"This certificate belongs to user X"
```

Common in Kubernetes cluster administration.

---

## ServiceAccount

Pods often need to communicate with the Kubernetes API.

For example:

```text
Pod
 │
 │ ServiceAccount credentials
 ▼
API Server
```

The ServiceAccount provides an identity for the workload.

Example:

```text
Pod
  │
  └── ServiceAccount
          │
          ▼
       API Server
```

---

## OIDC

OIDC allows Kubernetes to integrate with an external identity provider.

Conceptually:

```text
User
 │
 ▼
Identity Provider
 │
 ▼
OIDC token
 │
 ▼
Kubernetes API Server
```

For example, an organization may use an external identity platform for employee authentication.

You don't need to memorize the OIDC mechanics yet.

Remember:

> **OIDC = external identity provider integration.**

---

# 18. What's a Webhook?

A webhook basically means:

> "Kubernetes asks another service to help make a decision."

For example:

```text
API Server
    │
    │ "Is this token valid?"
    ▼
External Authentication Service
    │
    ▼
Yes / No
```

Or for admission:

```text
API Server
    │
    ▼
Admission Webhook
    │
    ▼
Allow / Modify / Reject
```

---

# 19. Now the built-in admission controllers

Your document lists:

```text
NamespaceLifecycle
LimitRanger
ResourceQuota
PodSecurity
NodeRestriction
StorageObjectInUseProtection
```

You don't need to memorize these yet.

Understand what they're doing.

### NamespaceLifecycle

Basically:

> "Don't allow operations that shouldn't happen in terminating/nonexistent namespaces."

---

### LimitRanger

Can provide/default resource limits or requests.

Example:

```text
User didn't specify CPU
       │
       ▼
LimitRanger
       │
       ▼
Default CPU value added
```

---

### ResourceQuota

Limits total resource usage in a namespace.

Example:

```text
Namespace quota:

CPU = 10 cores
Memory = 20 GB
```

If someone tries to exceed it:

```text
❌ Rejected
```

---

### PodSecurity

Applies Pod security policies/standards.

Conceptually:

```text
Pod
 │
 ▼
Security check
 │
 ├── Allowed
 └── Rejected
```

---

### NodeRestriction

Protects the boundary between kubelets and cluster resources.

Simplified:

> A kubelet shouldn't be able to arbitrarily pretend to be another node or modify resources it shouldn't control.

---

### StorageObjectInUseProtection

Helps prevent certain storage resources from being deleted while they are actively being used.

For example:

```text
Pod
 │
 ▼
PVC
 │
 ▼
Storage
```

Kubernetes takes precautions around deletion.

---

# 20. What the hell is "Aggregated API Layer"?

This is another thing you **do not need to worry about yet**.

The normal Kubernetes API handles resources like:

```text
Pods
Deployments
Services
Nodes
ConfigMaps
Secrets
```

But Kubernetes can be extended with additional APIs.

Conceptually:

```text
                 kube-apiserver
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
       Normal Kubernetes     Custom API
            APIs                 │
                                ▼
                         Custom API Server
```

---

# 21. What is `kubectl top`?

This is the example your document gives.

When you run:

```bash
kubectl top pods
```

Kubernetes needs metrics such as:

```text
CPU
Memory
```

These are commonly provided through the **Metrics API**, often backed by **metrics-server**.

Conceptually:

```text
kubectl top
    │
    ▼
metrics.k8s.io API
    │
    ▼
metrics-server
    │
    ▼
Node/Pod metrics
```

The aggregation mechanism allows additional API servers/services to expose Kubernetes-style APIs.

---

# 22. Why does Kubernetes need an API Server at all?

This is the big picture.

Without a central API:

```text
kubectl ─────► scheduler
kubectl ─────► controller
kubectl ─────► etcd
kubectl ─────► kubelet
```

That would be chaotic.

Instead:

```text
                   API Server
                /      |      \
               /       |       \
            etcd   Scheduler  Controllers
```

Everything has a controlled interface.

The API Server provides:

- Authentication
    
- Authorization
    
- Admission
    
- Validation
    
- API versioning
    
- Object management
    
- Communication endpoint
    
- Persistence interaction
    

That's why it is effectively the **front door of Kubernetes**.

---

# 23. Your original diagram translated into normal English

Your document says:

```text
Incoming Request
      │
      ▼
Authentication
      │
      ▼
Authorization
      │
      ▼
Admission
      │
      ▼
OpenAPI Validation
      │
      ▼
etcd Persistence
```

Read it as:

```text
                 Someone sends request
                         │
                         ▼
                 ┌─────────────────┐
                 │   WHO ARE YOU?  │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │ ARE YOU ALLOWED?│
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │ POLICY CHECKS   │
                 │ Modify/Reject?  │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │ IS IT VALID?    │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │ SAVE TO etcd    │
                 └─────────────────┘
```

That's what the entire section is trying to tell you.

---

# 24. One thing I would change in your notes

Don't try to memorize this:

> "kube-apiserver executes admission webhooks, serializes into protobuf, broadcasts watch events..."

yet.

Build your understanding in this order:

```text
LEVEL 1
What is API Server?
        ↓
Kubernetes front door

LEVEL 2
Authentication
        ↓
Who are you?

LEVEL 3
Authorization
        ↓
What can you do?

LEVEL 4
Admission
        ↓
Should this request be allowed/modified?

LEVEL 5
Validation
        ↓
Is the object structurally correct?

LEVEL 6
etcd
        ↓
Store the cluster state

LEVEL 7
Controllers/Scheduler
        ↓
Make the cluster actually do what was requested
```

Once those seven pieces are clear, terms like **X.509, OIDC, JWT, RBAC, admission webhook, OpenAPI, protobuf, watch events, API aggregation** become details rather than confusing architecture.

The **most important distinction** is:

```text
             kube-apiserver
                   │
        ┌──────────┴──────────┐
        │                     │
   "Can I accept             "What should
    this request?"             happen?"
        │                     │
        ▼                     ▼
 Authentication          Controllers
 Authorization           Scheduler
 Admission
 Validation
        │
        ▼
      etcd
```

**API Server decides whether the request can enter the Kubernetes system; controllers and scheduler are what turn the accepted desired state into actual running workloads.**


# 👾👾👾👾👾👾👾👾👾👾👾👾👾👾👾👾  2.3

This section is much easier if you stop thinking of etcd as some mysterious Kubernetes component.

## The simplest definition

> **etcd is Kubernetes' database.**

That's the first thing you need to understand.

```text
                    Kubernetes
                        │
                        ▼
                  kube-apiserver
                        │
                        ▼
                      etcd
                 "Cluster Database"
```

If Kubernetes needs to remember something about the cluster, **etcd is where that state is stored**.

---

# 1. What exactly does etcd store?

Suppose you create:

```bash
kubectl create deployment nginx --image=nginx --replicas=3
```

Kubernetes needs to remember:

```text
Deployment name = nginx
Desired replicas = 3
Image = nginx
```

It stores that cluster state in etcd.

Similarly, Kubernetes needs to remember things like:

```text
Pods
Deployments
Services
ConfigMaps
Secrets
Namespaces
Nodes
ServiceAccounts
Roles
RoleBindings
Leases
etc.
```

So you can think:

```text
                     etcd
                      │
       ┌──────────────┼──────────────┐
       ▼              ▼              ▼
   Deployments       Pods          Services
       │              │              │
       ▼              ▼              ▼
    Secrets        ConfigMaps     Namespaces
```

---

# 2. Why do they call it the "single source of truth"?

This phrase sounds fancy but means:

> **etcd contains the authoritative stored state of the Kubernetes cluster.**

Imagine you told Kubernetes:

```text
I want 3 nginx Pods.
```

The desired state gets stored.

```text
etcd:

nginx deployment
replicas = 3
```

Later, a controller checks reality:

```text
Desired:
3 Pods

Actual:
2 Pods
```

The controller says:

> "We're missing one."

It creates another Pod.

So:

```text
             etcd
              │
              │ Desired state
              ▼
        "I want 3 Pods"
              │
              ▼
         Controllers
              │
              ▼
        Actual cluster
```

---

# 3. Important: etcd does NOT run your containers

This is a common beginner misunderstanding.

etcd does **not** do this:

```text
etcd
 │
 └── run nginx ❌
```

Instead:

```text
etcd
 │
 └── stores information about nginx ✓
```

The actual container is run on a worker node:

```text
Worker Node
    │
   kubelet
    │
Container Runtime
    │
Container
```

So:

> **etcd remembers the desired/current cluster state. It doesn't execute your workloads.**

---

# 4. Why does etcd need Raft?

Now we get to the difficult-looking part.

Why not just have:

```text
etcd-1
```

?

Because if etcd-1 dies:

```text
etcd-1 ❌
```

Kubernetes loses its database.

So production Kubernetes often uses multiple etcd nodes.

For example:

```text
etcd-1
etcd-2
etcd-3
```

But now there's a problem.

Suppose someone changes something.

Which database is correct?

```text
etcd-1 → replicas = 3

etcd-2 → replicas = 4

etcd-3 → replicas = 3
```

We need the nodes to **agree**.

That's what **Raft** helps accomplish.

---

# 5. Raft in simple terms

Think of Raft as a system for making multiple etcd servers agree on the same information.

For example:

```text
             etcd cluster

          ┌─────────────┐
          │   etcd-1    │
          │   LEADER    │
          └──────┬──────┘
                 │
          ┌──────┴──────┐
          ▼             ▼
      etcd-2          etcd-3
     FOLLOWER        FOLLOWER
```

One node is elected:

> **Leader**

The others are:

> **Followers**

---

# 6. Why does etcd need a leader?

Suppose the API Server says:

```text
Change nginx replicas from 3 → 5.
```

The request goes to the etcd leader.

```text
API Server
     │
     │ "replicas = 5"
     ▼
etcd Leader
```

The leader records the change and replicates it.

```text
             Leader
               │
        ┌──────┴──────┐
        ▼             ▼
    Follower       Follower
```

Now the etcd members can maintain a consistent state.

---

# 7. What is this "log" they keep talking about?

This part:

> "Leader appends the write to its transaction log and replicates the log entry"

sounds complicated.

Think of a simple notebook.

The leader receives:

```text
Change replicas from 3 to 5
```

It records an operation:

```text
Log:

1. Create Deployment nginx
2. Set replicas = 3
3. Set replicas = 5
```

It replicates that operation to the other etcd members.

The log allows the distributed nodes to maintain the same ordered sequence of changes.

---

# 8. What is quorum?

This is one of the most important concepts in etcd.

Suppose you have **3 etcd nodes**:

```text
etcd-1
etcd-2
etcd-3
```

A majority is:

```text
2 out of 3
```

That's the quorum.

So:

```text
3 nodes
↓
Majority = 2
```

If the leader gets acknowledgement from enough members to form a majority, the entry can be committed.

---

# 9. Why does majority matter?

Suppose:

```text
etcd-1 = Leader
etcd-2 = Follower
etcd-3 = Follower
```

etcd-3 crashes:

```text
etcd-1 ✓
etcd-2 ✓
etcd-3 ❌
```

You still have:

```text
2 / 3
```

That's a majority.

The cluster can continue operating.

---

# 10. What if two nodes die?

```text
etcd-1 ✓
etcd-2 ❌
etcd-3 ❌
```

Now you have:

```text
1 / 3
```

No majority.

Therefore the cluster cannot safely continue making consensus-based writes.

This is deliberate.

It's better for etcd to stop accepting certain changes than to risk different nodes believing different things.

---

# 11. Why 3 or 5 nodes?

This explains this statement from your document:

> "3 nodes tolerating 1 failure or 5 nodes tolerating 2 failures."

### 3-node cluster

```text
3 nodes
Majority = 2
```

You can lose:

```text
1 node ✓
```

But not:

```text
2 nodes ❌
```

### 5-node cluster

```text
5 nodes
Majority = 3
```

You can lose:

```text
2 nodes ✓
```

But not:

```text
3 nodes ❌
```

---

# 12. Why not 4 nodes?

This is an excellent question.

With 4 nodes:

```text
4 nodes
Majority = 3
```

You can still tolerate only **1 failure**.

That's the same failure tolerance as 3 nodes.

So:

```text
3 nodes → tolerate 1 failure
4 nodes → tolerate 1 failure
5 nodes → tolerate 2 failures
```

Therefore adding a fourth node doesn't improve failure tolerance.

That's why odd numbers are commonly preferred for etcd clusters.

---

# 13. Now understand the diagram in your document

Your document has:

```text
Client Write
     │
     ▼
etcd LEADER
     │
     ├──────► Follower 1
     │
     └──────► Follower 2
```

Read it simply as:

```text
API Server says:

"Store this change."

       │
       ▼

     Leader
       │
       ├── copies change → Follower 1
       │
       └── copies change → Follower 2

       │
       ▼

Enough members agree
       │
       ▼
     COMMIT
```

---

# 14. Now the `/registry/` thing

Your document says:

```text
/registry/pods/{namespace}/{pod-name}
```

Don't worry about this too much.

Kubernetes organizes its stored objects using keys.

Conceptually:

```text
/registry/
    │
    ├── pods/
    │
    ├── deployments/
    │
    ├── services/
    │
    ├── secrets/
    │
    └── leases/
```

For example:

```text
/registry/pods/default/nginx-abc123
```

can represent a stored Pod object.

Think of it like a database table/key structure.

---

# 15. What does "protobuf" mean here?

Your document says:

> "All Kubernetes resources are serialized into Protocol Buffers."

Don't get distracted by this yet.

**Serialization** simply means:

> Convert an in-memory Kubernetes object into a format suitable for storage/transmission.

For example, Kubernetes has an object:

```text
Pod
 ├── name
 ├── namespace
 ├── image
 ├── ports
 └── resources
```

It needs to convert that object into a storable representation.

Protocol Buffers (**protobuf**) is one format Kubernetes uses internally.

For your Kubernetes architecture learning:

> **protobuf = data serialization format**

That's enough for now.

---

# 16. etcd backup

Now this part is extremely important for someone doing IT infrastructure.

Your document says:

> "Because losing etcd results in total cluster state loss..."

The idea is:

```text
Kubernetes
     │
     ▼
   etcd
     │
     └── cluster state
```

If you lose the etcd data and have no backup:

```text
Where are my Kubernetes objects?

❌ Deployments
❌ Services
❌ Configurations
❌ Secrets
❌ RBAC objects
❌ Cluster state
```

The containers that are already running may not immediately disappear just because etcd is unavailable, but **the cluster's authoritative control-plane state is lost/unavailable**, so recovery becomes a major problem.

That's why etcd backups matter.

---

# 17. What is an etcd snapshot?

Think of it as:

> **A backup of the etcd database.**

For example:

```text
Current etcd
     │
     ▼
Snapshot
     │
     ▼
etcd-backup.db
```

You can store that backup somewhere safe.

For example:

```text
etcd
 │
 ▼
Snapshot
 │
 ├── local backup
 └── remote backup
```

In production, you'd generally want the backup stored independently from the cluster itself.

---

# 18. Why does the document mention TLS certificates?

Because etcd is normally protected with TLS.

You don't want random machines doing:

```text
connect to etcd
```

The communication between Kubernetes control-plane components and etcd is secured.

So commands such as `etcdctl` often require appropriate certificates/keys.

Conceptually:

```text
etcdctl
   │
   │ TLS authentication
   ▼
etcd
```

---

# 19. Encryption at Rest

This section is very important.

Your document says:

> "Secrets stored in etcd are base64-encoded rather than encrypted by default."

This is a critical distinction.

Suppose you create:

```yaml
stringData:
  password: MyPassword123
```

Kubernetes Secret data is not automatically secure merely because it is represented as base64.

### Base64 is NOT encryption.

For example:

```text
hello
```

can be Base64 encoded:

```text
aGVsbG8=
```

Anyone can decode it.

So:

```text
Base64
≠
Encryption
```

---

# 20. Encryption at Rest

Kubernetes can be configured to encrypt sensitive resources before they are stored in etcd.

Conceptually:

```text
Secret
   │
   ▼
API Server
   │
   │ Encryption
   ▼
Encrypted data
   │
   ▼
etcd
```

Without encryption at rest:

```text
Secret
   │
   ▼
etcd
```

with the sensitive content stored in its configured representation.

With encryption:

```text
Secret
   │
   ▼
Encryption provider
   │
   ▼
Encrypted
   │
   ▼
etcd
```

The exact provider/configuration can vary, including KMS-based approaches.

---

# 21. Now the scary performance section

Your document suddenly jumps from:

> "etcd stores cluster state"

to:

> "fsync <10ms"

This is an **advanced operational topic**.

Don't mix it into your basic architecture understanding.

But here's what it means.

---

# 22. Why does etcd care about disk performance?

etcd needs to persist data safely.

Imagine:

```text
API Server
    │
    ▼
etcd
    │
    ▼
Disk
```

If the disk is extremely slow:

```text
Write
  │
  ▼
Disk
  │
  ├───────────────┐
  │               │
slow            slow
  │               │
  ▼               ▼
Raft timing problems
```

Because Raft depends on timely communication and durable writes, very slow storage can cause problems.

That's why etcd strongly prefers fast storage, particularly SSDs.

---

# 23. `heartbeat-interval`

This:

```text
--heartbeat-interval=100ms
```

is related to how frequently Raft leaders send heartbeats.

Conceptually:

```text
Leader
   │
   │ heartbeat
   ▼
Follower

Leader
   │
   │ heartbeat
   ▼
Follower
```

The followers use these communications to know the leader is still alive.

You don't need to memorize the exact default yet.

---

# 24. `election-timeout`

Now imagine:

```text
Leader
  │
  │ heartbeat
  │
  X
  │
Leader stops responding
```

Followers eventually conclude:

> "The leader may be dead."

Then a new election can occur.

```text
Old Leader ❌

Follower 1
    │
    ▼
Election
    │
    ▼
New Leader
```

The election timeout determines how long the system waits before triggering leader election under the relevant conditions.

Again, you don't need to memorize `1000ms` for basic Kubernetes learning.

---

# 25. `--quota-backend-bytes`

This is basically a size limit for the etcd backend database.

Think:

```text
etcd database
████████████████████████
          ↑
       capacity
```

If it gets too large, you need to manage it.

Your document says:

> crossing the limit puts the cluster into read-only mode.

More precisely, etcd can raise a **space quota exceeded alarm**, which prevents further writes until the condition is resolved.

This is an operational concern rather than something you need for your first understanding of Kubernetes architecture.

---

# 26. What is compaction?

This is another confusing word.

etcd keeps historical revisions of data.

Imagine:

```text
Deployment replicas:

Revision 1 → 1
Revision 2 → 2
Revision 3 → 3
Revision 4 → 5
Revision 5 → 3
```

Over time, keeping every old revision forever would consume space.

**Compaction** removes older historical revisions that are no longer needed.

Conceptually:

```text
Before:

Old history
████████████████████████████

       ↓ Compaction

After:

Recent useful history
██████████
```

So:

> **Compaction = clean up old revision history to reclaim storage.**

---

# 27. The entire etcd section in simple language

Your document:

> etcd is a strongly consistent distributed key-value database.

Means:

> **etcd is Kubernetes' database, and multiple etcd servers keep synchronized copies of that database.**

---

Your document:

> Raft consensus.

Means:

> **The etcd servers use Raft to agree on the same state.**

---

Your document:

> Single leader.

Means:

> **One etcd member is currently the Raft leader.**

---

Your document:

> Followers replicate log entries.

Means:

> **The leader sends changes to the other etcd members.**

---

Your document:

> Majority quorum.

Means:

> **A majority of the etcd members must participate for consensus to safely proceed.**

---

Your document:

> 3 nodes tolerate 1 failure.

Means:

```text
3 nodes
+
1 failure
=
still have 2
=
majority
```

---

Your document:

> Snapshot backup.

Means:

> **Take a backup of the etcd database so you can recover the cluster's state.**

---

Your document:

> Encryption at Rest.

Means:

> **Protect sensitive data stored in etcd instead of relying on Base64 representation.**

---

Your document:

> Compaction.

Means:

> **Clean up old etcd history to prevent unnecessary database growth.**

---

# 28. The architecture you should actually remember

Don't memorize all those flags yet.

Memorize this:

```text
                         Kubernetes
                              │
                              ▼
                       kube-apiserver
                              │
                              │
                              ▼
                    ┌──────────────────┐
                    │      etcd        │
                    │ Kubernetes DB    │
                    └────────┬─────────┘
                             │
                  ┌──────────┼──────────┐
                  ▼          ▼          ▼
               etcd-1     etcd-2     etcd-3
               Leader     Follower   Follower
                  │          │          │
                  └──────────┼──────────┘
                             │
                      Raft Consensus
```

And:

```text
               etcd
                │
        ┌───────┴────────┐
        │                │
   Desired State      Cluster State
        │                │
        └───────┬────────┘
                ▼
           Controllers
                │
                ▼
            Scheduler
                │
                ▼
           Worker Nodes
                │
                ▼
              Pods
```

---

# 29. One correction to keep in mind

The sentence:

> **"losing etcd results in total cluster state loss"**

is useful as a warning, but don't interpret it as:

> "If etcd goes down, every running container immediately disappears."

That's not what happens.

More accurately:

**etcd is the authoritative persistent store for Kubernetes control-plane state.** Losing its data without a backup can prevent you from recovering the cluster's Kubernetes objects and desired state, even though workloads may continue running temporarily while the control plane is unavailable.

That's an important distinction.

---

## Your learning priority

For your current Kubernetes learning level, I'd rank this section like this:

### Must understand now

```text
✓ etcd = Kubernetes database
✓ Stores cluster state
✓ API Server communicates with etcd
✓ Multiple etcd nodes provide HA
✓ Raft
✓ Leader
✓ Followers
✓ Quorum / majority
✓ 3 nodes → tolerate 1 failure
✓ 5 nodes → tolerate 2 failures
✓ etcd backup/snapshot
✓ Base64 ≠ encryption
```

### Learn later

```text
△ protobuf
△ /registry/ internals
△ fsync latency
△ heartbeat interval
△ election timeout
△ backend quota
△ compaction tuning
△ KMS encryption providers
△ detailed etcdctl restore procedure
```

The **one mental picture** to keep is:

> **API Server is the front door. etcd is the database behind it. Raft keeps multiple copies of that database consistent. One etcd member is the leader, the others follow, and a majority is required for safe consensus.**

# 👾👾👾👾👾👾👾👾👾👾👾👾👾👾👾👾👾  2.4

This section looks complicated because it mixes **one very simple job** with a lot of scheduling terminology.

## The actual job of kube-scheduler

Forget all the plugins for a moment.

> **kube-scheduler answers one question:**
> 
> **“This Pod needs to run. Which worker node should it go to?”**

It **does not create the container** and **does not start the Pod**.

Think:

```text
Pod created
   ↓
API Server
   ↓
"Pod needs a node"
   ↓
kube-scheduler
   ↓
"Put this Pod on worker-2"
   ↓
API Server
   ↓
spec.nodeName = worker-2
   ↓
worker-2's kubelet sees it
   ↓
kubelet tells container runtime to start it
```

So:

|Component|Job|
|---|---|
|API Server|Receives/stores the request|
|**Scheduler**|**Chooses the node**|
|Kubelet|Starts/manages the Pod on that node|
|Container runtime|Actually runs the containers|
|etcd|Stores the cluster state|

---

# 1. What is an "unbound Pod"?

This is the first important concept.

Suppose you create:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx
```

You didn't say which node should run it.

Therefore initially:

```yaml
spec:
  nodeName: ""
```

That's called an **unscheduled/unbound Pod**.

The scheduler sees:

```text
Pod nginx
nodeName = empty
        ↓
Scheduler:
"This Pod doesn't have a node."
        ↓
"I need to find one."
```

---

# 2. Imagine you have 3 worker nodes

```text
             Kubernetes Cluster

              Control Plane
                   │
              API Server
                   │
          ┌────────┴────────┐
          │                 │
       Worker-1          Worker-2
       CPU: 4             CPU: 8
       RAM: 8 GB         RAM: 16 GB
       │                   │
       Pods                Pods

                   Worker-3
                   CPU: 4
                   RAM: 8 GB
```

Now you create:

```text
nginx Pod
CPU request: 2
RAM request: 4 GB
```

Scheduler needs to decide:

```text
Worker-1 → Can it run there?
Worker-2 → Can it run there?
Worker-3 → Can it run there?
```

That's where **Filter** and **Score** come in.

---

# 3. FILTER = "Can this Pod run here?"

This is the most important distinction.

**Filter is a HARD RULE.**

The scheduler asks:

> "Is this node even eligible?"

Example:

```text
Worker-1
CPU available = 1

Pod requires = 2 CPU
```

Can't fit.

```text
Worker-1 ❌
```

Worker-2:

```text
CPU available = 6
Pod requires = 2

Worker-2 ✅
```

Worker-3:

```text
CPU available = 3
Pod requires = 2

Worker-3 ✅
```

Now:

```text
All nodes
   │
   ├── Worker-1 ❌
   ├── Worker-2 ✅
   └── Worker-3 ✅
```

The scheduler has eliminated Worker-1.

---

# 4. What are Filter Plugins?

They're simply **different rules used to eliminate nodes**.

For example:

### NodeResourcesFit

Checks:

```text
Does this node have enough CPU?
Does this node have enough RAM?
Does it have enough requested resources?
```

Example:

```text
Pod needs:
CPU = 4
RAM = 8 GB

Worker-1:
CPU available = 2
RAM available = 10 GB

Result → ❌
```

CPU isn't enough.

---

### NodeAffinity

You can tell Kubernetes:

> "This Pod must run on a node with this label."

For example:

```yaml
nodeSelector:
  disk: ssd
```

Nodes:

```text
Worker-1
disk=hdd       ❌

Worker-2
disk=ssd       ✅

Worker-3
disk=hdd       ❌
```

Only Worker-2 survives.

---

### Taint and Toleration

This one initially sounds horrible.

Think:

> **Taint = "Keep Pods away from this node unless they have permission."**

For example:

```text
Worker-1
taint: gpu=true
```

Normal Pod:

```text
"I don't tolerate gpu=true"

→ ❌ Worker-1
```

A Pod with the appropriate toleration:

```text
"I tolerate gpu=true"

→ ✅ Worker-1
```

This is commonly used for special-purpose nodes.

---

### VolumeBinding

Suppose your Pod needs a disk that is only available in a particular zone.

```text
Volume
   ↓
Zone A
```

Scheduler can't put the Pod in Zone B if that volume can't be used there.

So:

```text
Worker in Zone A → ✅
Worker in Zone B → ❌
```

---

# 5. After filtering, we have candidates

Suppose:

```text
Before filtering:

Worker-1
Worker-2
Worker-3
Worker-4

             ↓ FILTER

Worker-1 ❌
Worker-2 ✅
Worker-3 ✅
Worker-4 ❌
```

Now scheduler has:

```text
Worker-2
Worker-3
```

But it still has a problem:

> **Which one should I choose?**

That's where **Score** comes in.

---

# 6. SCORE = "Which eligible node is preferable?"

This is different from Filter.

Filter asks:

> **Can I use this node?**

Score asks:

> **Among the nodes I can use, which one should I prefer?**

For example:

```text
Worker-2 → 60 points
Worker-3 → 90 points
```

Scheduler chooses:

```text
Worker-3
```

So remember:

```text
FILTER = eliminate
SCORE  = rank
```

This one sentence will save you a lot of confusion.

---

# 7. Why have Score plugins?

Suppose both nodes are perfectly capable of running the Pod.

```text
Worker-2
CPU: 70% used

Worker-3
CPU: 20% used
```

Both technically work.

But you'd probably rather place the new Pod on Worker-3.

That's what scoring helps with.

---

## LeastAllocated

Basically:

> Prefer nodes with more available resources.

```text
Worker-2 → 80% used → lower score
Worker-3 → 30% used → higher score
```

---

## BalancedAllocation

Looks at resource balance.

Imagine:

```text
Worker-1:
CPU = 90% used
RAM = 20% used
```

Very unbalanced.

Another:

```text
Worker-2:
CPU = 50% used
RAM = 55% used
```

Much more balanced.

The scheduler can prefer Worker-2.

---

## ImageLocality

Imagine the Pod requires:

```text
nginx:1.27
```

Worker-1 already has the image:

```text
Worker-1
nginx:1.27 ✅
```

Worker-2 doesn't:

```text
Worker-2
nginx:1.27 ❌
```

Worker-1 can receive a higher score because the image is already there.

This can reduce image-pull work.

---

# 8. The entire scheduling process

Now the scary diagram becomes very simple.

```text
                 New Pod
                    │
                    ▼
          Is nodeName empty?
                    │
                   YES
                    │
                    ▼
              PRE-FILTER
        "Prepare information"
                    │
                    ▼
                FILTER
       "Which nodes are impossible?"
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
        Node-1    Node-2    Node-3
          ❌         ✅        ✅
                    │
                    ▼
                 SCORE
        "Which eligible node
          is preferable?"
                    │
             Node-2 = 70
             Node-3 = 90
                    │
                    ▼
              Choose Node-3
                    │
                    ▼
                RESERVE
          "Reserve/assume resources"
                    │
                    ▼
                  BIND
                    │
                    ▼
         API Server gets:
         
         spec.nodeName = node-3
```

Then the scheduler's job is essentially finished.

---

# 9. What happens after BIND?

This is **very important**.

The scheduler does **not** do:

```text
Scheduler
   ↓
start container
```

Instead:

```text
Scheduler
   ↓
API Server
   ↓
Pod now says:
nodeName = worker-3
   ↓
worker-3 kubelet sees the Pod
   ↓
Kubelet
   ↓
Container Runtime
   ↓
Container starts
```

So:

### Scheduler

**"WHERE should it run?"**

### Kubelet

**"Okay, it's assigned to my node. I need to make it run."**

### Container runtime

**"I'll actually create/start the container."**

---

# 10. What is Pre-Filter?

Don't overthink this.

Pre-Filter is basically:

> **"Before I start checking every node, let me prepare the information I need."**

For example, scheduler may determine:

```text
Pod needs:
CPU = 2
RAM = 4 GB

Pod has:
node affinity
volume requirements
topology requirements
etc.
```

Then it can efficiently evaluate nodes.

You don't need to memorize the internal implementation.

---

# 11. What is Post-Filter?

This is the interesting one.

Imagine:

```text
Worker-1 ❌
Worker-2 ❌
Worker-3 ❌
```

Every node failed.

Scheduler asks:

> "Can I somehow make room?"

One possibility is **preemption**.

Suppose:

```text
Worker-1

Existing Pod A
Priority = LOW

Existing Pod B
Priority = LOW
```

New Pod:

```text
Priority = HIGH
```

If removing a lower-priority Pod would make room:

```text
Remove lower-priority Pod
          ↓
Create space
          ↓
Schedule higher-priority Pod
```

That's **Pod preemption**.

You don't need to go deeply into this yet.

Just remember:

> **Post-Filter happens when normal filtering found no suitable node.**

---

# 12. What is Reserve?

This wording causes confusion.

Reserve doesn't mean:

> "The Pod has started."

It is more like:

> **"I've chosen this node, so temporarily account for the resources while I complete the scheduling operation."**

Think:

```text
Chosen node = Worker-2

Reserve:
"Okay, I'm going to use Worker-2."
```

Then Bind makes the actual assignment visible through the API server.

---

# 13. What is Bind?

This is the final step.

The scheduler effectively tells the API server:

```yaml
spec:
  nodeName: worker-2
```

Before:

```yaml
spec:
  containers:
    - name: nginx
      image: nginx

  nodeName: ""
```

After:

```yaml
spec:
  containers:
    - name: nginx
      image: nginx

  nodeName: worker-2
```

Now Kubernetes knows:

> **This Pod belongs on worker-2.**

---

# 14. Now the "6 phases" make sense

You can translate the entire documentation into this:

|Phase|Human translation|
|---|---|
|**Pre-Filter**|Prepare the information|
|**Filter**|Remove nodes that cannot work|
|**Post-Filter**|If nothing works, consider preemption|
|**Score**|Rank the nodes that can work|
|**Reserve**|Temporarily account for the selected node|
|**Bind**|Assign the Pod to that node|

The most important two are:

```text
FILTER → Can it run here?

SCORE  → Which eligible node should I prefer?
```

---

# 15. Node Affinity vs Pod Anti-Affinity

These names are also unnecessarily confusing.

### Node Affinity

You're talking about:

> **Pod → Node**

Example:

```text
"I want my database Pod on SSD nodes."
```

```text
Pod
 ↓
needs
 ↓
Node with disk=ssd
```

---

### Pod Anti-Affinity

You're talking about:

> **Pod → Other Pods**

Example:

You have:

```text
API-Pod-1
API-Pod-2
API-Pod-3
```

You don't want all of them on the same node.

Why?

If that node dies:

```text
Worker-1 dies

API-1 ❌
API-2 ❌
API-3 ❌
```

Instead:

```text
Worker-1 → API-1
Worker-2 → API-2
Worker-3 → API-3
```

Now one node failure doesn't take all replicas with it.

So:

```text
Node Affinity
= "I want a certain TYPE of node."

Pod Anti-Affinity
= "I don't want my Pods together with certain other Pods."
```

---

# 16. Topology Spread Constraints

This is basically a more controlled version of:

> **"Spread my Pods across different locations."**

For example:

```text
Zone A          Zone B          Zone C

Worker-1        Worker-3        Worker-5
Worker-2        Worker-4        Worker-6

API-1           API-2           API-3
```

Instead of:

```text
Zone A
API-1
API-2
API-3

Zone B
nothing

Zone C
nothing
```

Topology spread helps you control this distribution.

`topologyKey` tells Kubernetes **what kind of location you're talking about**, such as:

```text
kubernetes.io/hostname
```

meaning individual nodes, or a zone label such as:

```text
topology.kubernetes.io/zone
```

meaning availability zones.

---

# 17. `required` vs `preferred`

This is another important distinction.

### Required

```text
requiredDuringSchedulingIgnoredDuringExecution
```

Means:

> **"This condition must be satisfied."**

If not:

```text
Node ❌
```

No compromise.

---

### Preferred

```text
preferredDuringSchedulingIgnoredDuringExecution
```

Means:

> **"I'd like this, but it's not mandatory."**

For example:

```text
Prefer SSD nodes
```

If SSD nodes are available:

```text
SSD → preferred
```

But if no SSD node is available, Kubernetes can potentially use another eligible node.

Therefore:

```text
REQUIRED  = MUST
PREFERRED = NICE TO HAVE
```

---

# The mental model you should keep

Don't memorize all those plugin names yet.

Remember this:

```text
                 POD
                  │
                  ▼
          kube-scheduler
                  │
         "Where should it go?"
                  │
                  ▼
          ┌───────────────┐
          │    FILTER     │
          │               │
          │ Remove nodes  │
          │ that CANNOT   │
          │ run the Pod   │
          └───────┬───────┘
                  │
            Eligible nodes
                  │
                  ▼
          ┌───────────────┐
          │     SCORE     │
          │               │
          │ Rank nodes    │
          │ that CAN run  │
          │ the Pod       │
          └───────┬───────┘
                  │
                  ▼
             Pick node
                  │
                  ▼
               BIND
                  │
                  ▼
          API Server:
     nodeName = worker-2
                  │
                  ▼
        worker-2 kubelet
                  │
                  ▼
        Container Runtime
                  │
                  ▼
             Container
```

### What you actually need to know right now

**Must know:**

1. Scheduler chooses **which node** gets a Pod.
    
2. It watches for Pods that don't have a `nodeName`.
    
3. **Filter = hard rules / eliminate nodes.**
    
4. **Score = rank eligible nodes.**
    
5. **Bind = assign the Pod to the chosen node.**
    
6. Scheduler **does not start containers**.
    
7. Kubelet + container runtime handle starting the workload.
    
8. Node affinity = rules about **which nodes**.
    
9. Pod anti-affinity = rules about **which other Pods should/shouldn't be colocated**.
    
10. Topology spread = control **how Pods are distributed across nodes/zones**.
    

The plugin names such as `NodeResourcesFit`, `LeastAllocated`, `BalancedAllocation`, etc. are **implementation details of those decisions**. You can learn them after the core scheduling flow is clear.

# 👾👾👾👾👾👾👾👾👾👾👾👾👾👾👾👾👾  2.5

This is another section where the documentation makes something relatively simple look terrifying.

The **kube-controller-manager** can be understood with one idea:

> **Controllers continuously look at Kubernetes and fix things when reality doesn't match what you asked for.**

That's it.

---

# 1. First: what is a "controller"?

Suppose you tell Kubernetes:

```yaml
replicas: 3
```

You are saying:

> "I want 3 copies of this application."

But Kubernetes has to continuously make sure that remains true.

Imagine this:

```text
Desired state:
3 Pods

Actual state:
3 Pods
```

Everything is fine.

Then one Pod crashes:

```text
Desired:
3 Pods

Actual:
2 Pods
```

Something needs to notice this.

That's the **ReplicaSet Controller**.

It sees:

```text
I want: 3
I have: 2

Difference = 1
```

So it creates another Pod.

```text
2 Pods
   ↓
ReplicaSet Controller
   ↓
Create 1 Pod
   ↓
3 Pods
```

That's a **control loop**.

---

# 2. The basic controller loop

Forget "30+ goroutines" for now.

Think:

```text
        ┌──────────────────────┐
        │   Desired State      │
        │      replicas: 3     │
        └──────────┬───────────┘
                   │
                   ▼
              Controller
                   │
                   ▼
          Check actual state
                   │
                   ▼
        ┌──────────────────────┐
        │ Desired = 3          │
        │ Actual  = 2          │
        └──────────┬───────────┘
                   │
                   ▼
             "We're missing 1"
                   │
                   ▼
              Take action
                   │
                   ▼
              Create Pod
                   │
                   ▼
           Check again...
                   │
                   └──────────────┐
                                  │
                                  ▼
                         Repeat forever
```

This is called:

> **Reconciliation**

You will hear this word constantly in Kubernetes.

---

# 3. What does "reconcile" actually mean?

Don't let the word scare you.

**Reconcile = make actual state match desired state.**

For example:

```text
Desired state = 5 Pods
Actual state  = 3 Pods
```

Controller:

```text
3 → 5
```

Or:

```text
Desired state = 2 Pods
Actual state  = 5 Pods
```

Controller:

```text
5 → 2
```

Or:

```text
Desired state = 3
Actual state = 3
```

Controller:

```text
Nothing to do.
```

That's reconciliation.

---

# 4. Where does desired state come from?

This connects directly to the **API Server + etcd** section you just learned.

For example:

```text
You
 │
 │ kubectl apply
 ▼
API Server
 │
 ▼
etcd
 │
 │ Desired state stored
 ▼
Controllers
```

So you can think:

```text
etcd
 ↓
"What does Kubernetes want?"

Actual cluster
 ↓
"What is actually happening?"
```

Controller:

```text
"These two don't match.
I'll fix it."
```

---

# 5. Why are there 30+ controllers?

Because there are **many different things Kubernetes needs to continuously manage**.

For example:

```text
ReplicaSet Controller
    → maintain number of Pods

Deployment Controller
    → manage application rollouts

StatefulSet Controller
    → manage stateful applications

DaemonSet Controller
    → maintain Pods across nodes

Job Controller
    → run tasks to completion

Node Controller
    → monitor nodes

Namespace Controller
    → handle namespace deletion

EndpointSlice Controller
    → maintain Service backend information

PVC Controller
    → handle storage binding
```

Instead of one giant program doing everything, Kubernetes has many specialized controllers.

---

# 6. Think of controller-manager as a manager with many employees

This is a useful mental model.

```text
             kube-controller-manager
                       │
       ┌───────────────┼────────────────┐
       │               │                │
       ▼               ▼                ▼
 ReplicaSet         Deployment       Node
 Controller         Controller       Controller

       │               │                │
       ▼               ▼                ▼
 "Keep 3 Pods"     "Manage rollout"  "Is node alive?"

       ┌───────────────┼────────────────┐
       ▼               ▼                ▼
  StatefulSet       DaemonSet          Job
  Controller        Controller       Controller
```

They're all inside the **kube-controller-manager process**, but they perform different jobs.

---

# 7. ReplicaSet Controller

This one is the easiest.

Suppose:

```yaml
spec:
  replicas: 3
```

Controller checks:

```text
Desired = 3
Actual = 3
```

Nothing happens.

Now:

```text
Pod-1 ✅
Pod-2 ❌
Pod-3 ✅
```

Actual = 2.

Controller:

```text
Desired = 3
Actual = 2

Need 1 more Pod.
```

It creates one.

---

### What if there are too many?

Suppose:

```text
Desired = 3
Actual = 4
```

Controller removes an excess Pod.

So:

```text
Actual < Desired
    → create

Actual > Desired
    → remove

Actual = Desired
    → do nothing
```

This is one of the most important examples of reconciliation.

---

# 8. Deployment Controller

Now things become one level more interesting.

A Deployment doesn't directly manage individual Pods.

It works through **ReplicaSets**.

Think:

```text
Deployment
     │
     ▼
ReplicaSet
     │
     ▼
Pods
```

Suppose you're running:

```text
nginx:v1
```

with:

```text
3 replicas
```

You update it to:

```text
nginx:v2
```

The Deployment Controller manages the transition.

Conceptually:

```text
Old ReplicaSet
nginx:v1
   │
   │ slowly reduce
   ▼
Pods: 3 → 2 → 1 → 0

New ReplicaSet
nginx:v2
   │
   │ slowly increase
   ▼
Pods: 0 → 1 → 2 → 3
```

The exact rollout behavior depends on the Deployment strategy and settings such as:

```text
maxSurge
maxUnavailable
```

But the important idea is:

> **Deployment Controller manages the rollout by controlling ReplicaSets.**

---

# 9. StatefulSet Controller

This is for applications where Pods need **identity and stable storage/network identity**.

Normal Deployment:

```text
Pod names:

random-name-a
random-name-b
random-name-c
```

StatefulSet:

```text
database-0
database-1
database-2
```

These identities matter.

For example, a database cluster might need:

```text
database-0
database-1
database-2
```

The StatefulSet controller manages these Pods according to StatefulSet rules.

A common default behavior is ordered creation:

```text
Create database-0
      ↓
Wait until ready
      ↓
Create database-1
      ↓
Wait until ready
      ↓
Create database-2
```

Scaling down generally happens in reverse order:

```text
database-2
    ↓
database-1
    ↓
database-0
```

Don't worry about every StatefulSet detail yet.

Remember:

> **Deployment = interchangeable Pods**
> 
> **StatefulSet = Pods with stable identity/stateful behavior**

---

# 10. DaemonSet Controller

This one is actually very easy.

Suppose you want a logging agent on **every worker node**.

```text
Worker-1 → logging Pod
Worker-2 → logging Pod
Worker-3 → logging Pod
Worker-4 → logging Pod
```

You don't say:

```text
replicas: 4
```

Instead, you say:

> "Run this Pod on every eligible node."

The DaemonSet Controller watches nodes.

New node joins:

```text
Worker-5 joins
       ↓
DaemonSet Controller notices
       ↓
Creates logging Pod on Worker-5
```

Node disappears:

```text
Worker-5 removed
       ↓
Controller adjusts state
```

Typical uses include:

- log collection agents
    
- monitoring agents
    
- networking components
    

---

# 11. Job Controller

A Job is different from a Deployment.

Deployment:

> "Keep my application running."

Job:

> "Run this task and finish it."

For example:

```text
Process 10,000 records
```

You don't want:

```text
Run forever
```

You want:

```text
Start
 ↓
Process records
 ↓
Finish
```

The Job Controller watches this.

If a Pod fails:

```text
Job
 ↓
Pod fails
 ↓
Controller notices
 ↓
Create/retry another Pod
```

`backoffLimit` controls how many retries are allowed before the Job is considered failed.

---

# 12. CronJob Controller

CronJob is basically:

> **"Create Jobs according to a schedule."**

For example:

```text
Every day at 2 AM
       ↓
Create Job
       ↓
Job creates Pod
       ↓
Task runs
       ↓
Task finishes
```

So:

```text
CronJob
   ↓
Job
   ↓
Pod
```

---

# 13. Node Controller

This one is important for IT support because it deals with node health.

Imagine:

```text
Worker-1
    │
    │ kubelet heartbeat
    ▼
Control Plane
```

The kubelet periodically tells Kubernetes:

> "I'm still here."

The Node Controller watches node health information.

If a node stops responding:

```text
Worker-1
   X
No heartbeat / lease updates
   ↓
Node Controller notices
   ↓
Node becomes NotReady
```

Eventually Kubernetes may take additional actions such as tainting the node and handling Pods associated with it.

### Important correction to the pasted documentation

Don't memorize **"40 seconds = NotReady, 5 minutes = eviction"** as universal constants.

Those timings depend on Kubernetes configuration and behavior, and eviction behavior has additional conditions. The important concept is:

> **Node Controller detects unhealthy/unreachable nodes and takes actions to protect/reconcile the cluster.**

---

# 14. EndpointSlice Controller

This one sounds much worse than it is.

Suppose you have:

```text
Service: my-app
```

and three Pods:

```text
Pod-1 → 10.0.0.10
Pod-2 → 10.0.0.11
Pod-3 → 10.0.0.12
```

The Service needs to know which Pods are available as backends.

The EndpointSlice system maintains information like:

```text
my-app Service
      │
      ▼
EndpointSlice
      │
      ├── 10.0.0.10
      ├── 10.0.0.11
      └── 10.0.0.12
```

If Pod-2 becomes unavailable:

```text
Pod-2 ❌
```

the endpoint information is updated accordingly.

So remember:

> **EndpointSlice Controller keeps track of which Pod IPs are available behind Services.**

---

# 15. HPA Controller

HPA = **Horizontal Pod Autoscaler**.

This one changes the number of replicas based on metrics.

Suppose:

```text
Deployment
replicas = 3
```

CPU usage becomes high.

HPA might determine:

```text
3 replicas aren't enough
       ↓
Increase to 5
```

Later load decreases:

```text
5 replicas
    ↓
Load decreases
    ↓
Reduce to 3
```

Conceptually:

```text
                 Metrics
                    │
                    ▼
              HPA Controller
                    │
          "How many replicas
             do we need?"
                    │
                    ▼
              Deployment
                    │
                    ▼
               ReplicaSet
                    │
                    ▼
                  Pods
```

### One correction

The documentation says the HPA controller queries the Metrics API every **15 seconds**. That's the default sync period in common Kubernetes configurations, but it is configurable. Don't treat 15 seconds as a fundamental Kubernetes law.

---

# 16. PVC Binder Controller

Now storage.

You learned about PVC/PV earlier.

Think:

```text
PVC
"I need 100 GB"
```

PV:

```text
PV-1 → 50 GB
PV-2 → 100 GB
PV-3 → 500 GB
```

The controller looks for a suitable PV.

```text
PVC: 100 GB
       ↓
PVC Binder Controller
       ↓
PV-2: 100 GB
       ↓
Bind them
```

Conceptually:

```text
PVC
 ↓
"I need storage"

Controller
 ↓
"Which PV satisfies this?"

 ↓

PV
 ↓
"Okay, you're bound."
```

---

# 17. Namespace Controller

Suppose you delete:

```text
namespace: development
```

There may be hundreds of objects inside it:

```text
development
 ├── Pods
 ├── Services
 ├── ConfigMaps
 ├── Secrets
 ├── Jobs
 └── PVCs
```

The Namespace Controller helps handle the cleanup process.

Conceptually:

```text
Delete Namespace
       ↓
Namespace Controller
       ↓
Clean up objects
       ↓
Handle finalizers
       ↓
Namespace disappears
```

---

# 18. The BIG picture

Now connect everything you've learned so far.

Suppose you run:

```bash
kubectl create deployment nginx --image=nginx --replicas=3
```

The flow is roughly:

```text
                    kubectl
                       │
                       ▼
                  API Server
                       │
                       ▼
                     etcd
                       │
                       │ Desired state:
                       │ "I want nginx = 3"
                       ▼
            Deployment Controller
                       │
                       ▼
                 ReplicaSet
                       │
                       ▼
              "Need 3 Pods"
                       │
                       ▼
              Create Pods
                       │
                       ▼
            Scheduler sees Pods
            without nodeName
                       │
                       ▼
              Chooses nodes
                       │
                       ▼
              Bind Pods to nodes
                       │
                       ▼
                 Kubelets
                       │
                       ▼
             Container Runtime
                       │
                       ▼
                  Containers
```

This is the architecture you should be building in your head.

---

# 19. What does "control loop" mean in this entire picture?

Let's say you asked for:

```text
3 nginx Pods
```

Kubernetes doesn't just say:

> "Okay, done."

It continuously checks.

```text
              DESIRED
                │
                │
                ▼
        "3 nginx Pods"
                │
                ▼
          Controller
                │
                ▼
           ACTUAL STATE
                │
       ┌────────┴────────┐
       │                 │
      3 Pods            2 Pods
       │                 │
       ▼                 ▼
      GOOD          Create another
```

Then later:

```text
3 Pods
 ↓
Pod crashes
 ↓
2 Pods
 ↓
Controller notices
 ↓
Create replacement
 ↓
3 Pods
```

This happens continuously.

That's why it's called a **control loop**.

---

# 20. Don't confuse Controller Manager with Scheduler

This is extremely important because you're learning them one after another.

### Scheduler

Asks:

> **"Which node should this Pod go to?"**

```text
Pod
 ↓
Scheduler
 ↓
Worker-2
```

### Controller

Asks:

> **"Does the cluster currently match what we requested?"**

```text
Desired = 3 Pods
Actual = 2 Pods
       ↓
Controller
       ↓
Create another Pod
```

So:

```text
Scheduler
= WHERE should the Pod run?

Controller
= WHAT should exist, and is it actually there?
```

And:

```text
Kubelet
= Make the assigned Pod actually run on this node.
```

---

# 21. The three concepts together

This is probably the most useful diagram for your current Kubernetes learning:

```text
                 API SERVER
                     │
                     ▼
                   etcd
             "Desired State"
                     │
                     ▼
          ┌────────────────────┐
          │ Controller Manager │
          └─────────┬──────────┘
                    │
             "We need 3 Pods"
                    │
                    ▼
                  Pods
             nodeName = empty
                    │
                    ▼
              kube-scheduler
                    │
              "Which node?"
                    │
              ┌─────┴─────┐
              ▼           ▼
           Worker-1    Worker-2
              ❌           ✅
                          │
                          ▼
                  nodeName=worker-2
                          │
                          ▼
                       kubelet
                          │
                          ▼
                 container runtime
                          │
                          ▼
                      Container
```

And then the loop continues:

```text
Container crashes
       ↓
Actual state changes
       ↓
Controller notices
       ↓
Reconciliation
       ↓
Replacement Pod
       ↓
Scheduler chooses node
       ↓
Kubelet starts it
```

## What you should remember from 2.5

Don't try to memorize 30+ controllers yet.

Remember these:

|Component|Simple meaning|
|---|---|
|**Controller Manager**|Collection of controllers that continuously reconcile the cluster|
|**Controller**|Watches something and takes action when reality differs from desired state|
|**ReplicaSet Controller**|Maintains desired number of Pods|
|**Deployment Controller**|Manages ReplicaSets and rollouts|
|**StatefulSet Controller**|Manages stateful Pods with stable identity|
|**DaemonSet Controller**|Ensures a Pod runs on eligible nodes|
|**Job Controller**|Makes batch tasks complete|
|**CronJob Controller**|Creates Jobs on a schedule|
|**Node Controller**|Handles node health/state|
|**HPA Controller**|Adjusts replica count based on metrics|
|**PVC Controller**|Helps bind storage claims to volumes|
|**EndpointSlice Controller**|Maintains Service backend endpoint information|
|**Namespace Controller**|Handles namespace cleanup|

The **single sentence** to remember is:

> **The controller-manager is Kubernetes' continuous "make reality match what I asked for" system.**

That concept is much more important than memorizing the 30+ individual controllers.