# Kubernetes Learning & Hands-On Practice

This repository documents my hands-on Kubernetes learning journey, including concepts, commands, YAML manifests, troubleshooting, and interview preparation.

My focus is on understanding **how Kubernetes works in practice**, rather than only learning theoretical concepts.

---

## 🎯 Learning Objective

I am learning Kubernetes with a focus on:

* Kubernetes fundamentals
* Pods
* Deployments
* ReplicaSets
* Services
* NodePort
* Labels and Selectors
* Container networking
* Docker image deployment
* Scaling
* Rolling updates
* Troubleshooting Kubernetes workloads
* Kubernetes commands (`kubectl`)
* AKS fundamentals
* Kubernetes concepts from an interview perspective

---

# 1. Docker Image → Kubernetes Deployment

I started with an existing Docker image hosted on Docker Hub:

```text
amitverma5/llp:latest
```

The objective was to deploy this image into Kubernetes.

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: express

spec:
  replicas: 2

  selector:
    matchLabels:
      apps: express

  template:
    metadata:
      labels:
        apps: express

    spec:
      containers:
        - name: express
          image: amitverma5/llp:latest
          ports:
            - containerPort: 3000
```

### Commands practiced

```bash
kubectl apply -f deployment-nodeapp.yaml
```

```bash
kubectl get deployments
```

```bash
kubectl get pods
```

```bash
kubectl get pods -o wide
```

---

# 2. Kubernetes Pods

The Deployment was configured with:

```yaml
replicas: 2
```

Kubernetes therefore created two Pods running the Express application.

Example:

```text
express-699fb568fb-7d6fk
express-699fb568fb-7pjhg
```

Both Pods reached:

```text
1/1 Running
```

### Interview Understanding

A **Pod** is the smallest deployable unit in Kubernetes.

A Pod can contain one or more containers.

In this project, each Pod contains one Express application container.

---

# 3. Service

Pods are ephemeral, so I created a Kubernetes Service to provide stable networking.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: express-service

spec:
  type: NodePort

  selector:
    apps: express

  ports:
    - port: 3000
      targetPort: 3000
      nodePort: 30080
```

### Service configuration

```text
Service Port  = 3000
Target Port   = 3000
NodePort      = 30080
```

The request flow is:

```text
Client
   |
   | :30080
   ↓
NodePort
   |
   ↓
Service :3000
   |
   ↓
Pod :3000
   |
   ↓
Express Application
```

---

# 4. Important Kubernetes Concept: Labels and Selectors

One of the issues I encountered was a Service showing:

```text
ENDPOINTS: <none>
```

I investigated the Pod labels:

```bash
kubectl get pods --show-labels
```

The Pods had:

```text
apps=express
```

But my Service was configured with:

```yaml
selector:
  app: express
```

The problem was:

```text
Pod label:
apps=express

Service selector:
app=express
```

These do not match.

### Fix

I changed the Service selector to:

```yaml
selector:
  apps: express
```

After applying the change:

```bash
kubectl apply -f ex-service.yaml
```

the Service successfully discovered the Pods.

### Interview Learning

Kubernetes Service discovery depends on matching:

```text
Pod labels
      ↕
Service selector
```

Even a small difference such as:

```text
app
```

vs

```text
apps
```

will prevent the Service from selecting the Pods.

---

# 5. Troubleshooting CrashLoopBackOff

During deployment, my Pods initially showed:

```text
0/1 CrashLoopBackOff
```

I checked the application logs:

```bash
kubectl logs deployment/express
```

The error was:

```text
exec /usr/local/bin/docker-entrypoint.sh:
exec format error
```

This indicated a container architecture problem.

My Kubernetes node was:

```text
amd64
```

The Docker image had been built for a different architecture.

### Solution

I rebuilt the Docker image for AMD64:

```bash
docker buildx build \
  --platform linux/amd64 \
  -t amitverma5/llp:latest \
  --push .
```

Then I restarted the Deployment:

```bash
kubectl rollout restart deployment express
```

After that:

```bash
kubectl get pods
```

showed:

```text
1/1 Running
1/1 Running
```

### Interview Learning

When a Pod enters:

```text
CrashLoopBackOff
```

I should not immediately change the YAML.

First investigate:

```bash
kubectl logs <pod-name>
```

and:

```bash
kubectl describe pod <pod-name>
```

Common causes include:

* Application crash
* Incorrect startup command
* Missing environment variables
* Incorrect image
* Architecture mismatch
* Configuration errors
* Dependency/database problems

---

# 6. Testing the Application

After fixing the Deployment and Service, I tested the application using:

```bash
curl localhost:30080
```

The response was:

```html
<h1>Express</h1>
<p>Welcome to Express</p>
```

This confirmed that the request was successfully reaching the Express application through Kubernetes.

---

# 7. `kubectl create` vs `kubectl apply`

I also learned the difference between:

```bash
kubectl create -f file.yaml
```

and:

```bash
kubectl apply -f file.yaml
```

When the Service already existed, running:

```bash
kubectl create -f ex-service.yaml
```

returned:

```text
AlreadyExists
```

### Understanding

`kubectl create`:

```text
Create a new resource
```

`kubectl apply`:

```text
Create the resource if it doesn't exist
OR
Update the existing resource
```

For configuration managed through YAML files, I will generally use:

```bash
kubectl apply -f <file>.yaml
```

---

# 8. Scaling

I practiced changing the number of application replicas.

Scale up:

```bash
kubectl scale deployment express --replicas=5
```

Check:

```bash
kubectl get pods
```

Scale down:

```bash
kubectl scale deployment express --replicas=2
```

### Interview Question

**What happens when replicas are changed?**

The Deployment updates the desired state, and Kubernetes creates or removes Pods to match the requested replica count.

---

# 9. Kubernetes Commands I Am Practicing

## Cluster Information

```bash
kubectl get nodes
```

```bash
kubectl get nodes -o wide
```

## Deployments

```bash
kubectl get deployments
```

```bash
kubectl describe deployment express
```

## Pods

```bash
kubectl get pods
```

```bash
kubectl get pods -o wide
```

```bash
kubectl get pods --show-labels
```

```bash
kubectl describe pod <pod-name>
```

## Logs

```bash
kubectl logs <pod-name>
```

```bash
kubectl logs deployment/express
```

## Services

```bash
kubectl get svc
```

```bash
kubectl describe svc express-service
```

## Apply Configuration

```bash
kubectl apply -f deployment.yaml
```

```bash
kubectl apply -f service.yaml
```

## Scaling

```bash
kubectl scale deployment express --replicas=5
```

## Restart

```bash
kubectl rollout restart deployment express
```

---

# 10. Problems I Encountered

This section is important because I want to document **real troubleshooting experience**, not only successful commands.

| Problem                                 | What I learned                                                | Solution                                           |
| --------------------------------------- | ------------------------------------------------------------- | -------------------------------------------------- |
| `kind: Deployments` error               | Kubernetes resource kinds are case-sensitive and singular     | Changed to `kind: Deployment`                      |
| `matchLables` / `lables`                | YAML/Kubernetes field names must be exact                     | Changed to `matchLabels` / `labels`                |
| `CrashLoopBackOff`                      | Always inspect logs and Pod events                            | Used `kubectl logs` and `kubectl describe`         |
| `exec format error`                     | Container architecture must match node architecture           | Rebuilt image for `linux/amd64`                    |
| Service `ENDPOINTS <none>`              | Service selector must match Pod labels                        | Changed `app` to `apps`                            |
| `AlreadyExists`                         | `kubectl create` cannot update an existing resource           | Used `kubectl apply`                               |
| `curl localhost:30080` initially failed | NodePort access depends on the Kubernetes environment/network | Tested using Kubernetes networking/port forwarding |

---

# 11. Interview Preparation

## Questions I Should Be Able to Answer

### Kubernetes Fundamentals

* What is Kubernetes?
* What is a Pod?
* What is a Deployment?
* What is a ReplicaSet?
* What is a Service?
* Why do we need a Service?
* What is NodePort?
* What is ClusterIP?
* What is LoadBalancer?
* What is a Namespace?

### Deployment

* How do you deploy a Docker image to Kubernetes?
* How does Kubernetes pull an image from Docker Hub?
* What happens when a Pod crashes?
* What happens when a node goes down?
* How does a Deployment maintain replicas?
* How do you scale a Deployment?
* How do you update an application image?
* How do you perform a rollback?

### Networking

* What is a Kubernetes Service?
* What is a Service selector?
* What are labels?
* What is `targetPort`?
* What is `port`?
* What is `nodePort`?
* Why would a Service show `ENDPOINTS <none>`?
* How does traffic reach a Pod?

### Troubleshooting

I should be comfortable with:

```bash
kubectl get pods
kubectl describe pod <pod>
kubectl logs <pod>
kubectl get svc
kubectl describe svc <service>
kubectl get endpointslice
kubectl get deployment
kubectl describe deployment <deployment>
```

---

# 12. Current Hands-On Architecture

```text
                    Docker Hub
                        |
                        | Pull Image
                        ↓
                Kubernetes Cluster
                        |
                  Deployment
                  replicas: 2
                        |
              ┌─────────┴─────────┐
              ↓                   ↓
          Pod 1                 Pod 2
       Express:3000          Express:3000
              ↑                   ↑
              └─────────┬─────────┘
                        |
                 Service
              express-service
                  :3000
                        |
                    NodePort
                    :30080
                        |
                       curl
```

---

# 13. Next Kubernetes Topics

My next hands-on topics are:

* [ ] ReplicaSet
* [ ] ClusterIP
* [ ] NodePort
* [ ] LoadBalancer
* [ ] Namespaces
* [ ] ConfigMap
* [ ] Secrets
* [ ] Environment variables
* [ ] Resource requests and limits
* [ ] Liveness probes
* [ ] Readiness probes
* [ ] Rolling updates
* [ ] Rollbacks
* [ ] Persistent Volumes
* [ ] Persistent Volume Claims
* [ ] Ingress
* [ ] Helm
* [ ] Kubernetes networking
* [ ] AKS
* [ ] Kubernetes troubleshooting
* [ ] CI/CD deployment to Kubernetes

---

# 14. Goal

My goal is to move from simply knowing Kubernetes commands to being able to explain and troubleshoot a complete application deployment:

```text
Developer
   ↓
Git
   ↓
Docker Build
   ↓
Docker Hub / Container Registry
   ↓
Kubernetes
   ↓
Deployment
   ↓
Pods
   ↓
Service
   ↓
Application
```

I am documenting both **successful implementations and failures** because troubleshooting is an important part of real-world Kubernetes and DevOps work.
