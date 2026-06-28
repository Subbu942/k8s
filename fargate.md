# Deploying a Web Application on AWS EKS using Fargate

## What is AWS Fargate?

**AWS Fargate** is a **serverless compute engine** for containers. It allows you to run containers without provisioning or managing EC2 instances.

When using **Amazon EKS with Fargate**:

- AWS manages the Kubernetes Control Plane.
- AWS also manages the Worker Nodes (you don't create EC2 instances).
- You only deploy your Kubernetes applications.
- AWS automatically provisions the compute resources needed to run your Pods.

> **Think of it this way:**
>
> - **EKS + EC2:** You manage Worker Nodes.
> - **EKS + Fargate:** AWS manages everything except your application.

---

# EKS with EC2 vs EKS with Fargate

| EKS with EC2 | EKS with Fargate |
|--------------|------------------|
| You create EC2 Worker Nodes | No EC2 instances |
| You manage node scaling | AWS manages compute automatically |
| You patch worker nodes | AWS patches infrastructure |
| You choose EC2 instance types | AWS provisions compute dynamically |
| Better for high-performance workloads | Best for small, event-driven, or microservices |

---

# Architecture

```
                     AWS Cloud

            +------------------------+
            |   Amazon EKS Cluster   |
            | (Managed Control Plane)|
            +------------------------+
                       │
                       │
             Kubernetes Scheduler
                       │
        +--------------+--------------+
        |                             |
        ▼                             ▼
   Fargate Pod                  Fargate Pod
 (No EC2 Instance)          (No EC2 Instance)
        │                             │
        └──────────────┬──────────────┘
                       ▼
                 Your Application
```

Unlike EC2-based clusters, there are **no Worker Nodes** visible to you.

---

# Deployment Workflow

```
Application Source Code
          │
          ▼
Build Docker Image
          │
          ▼
Push Image to Amazon ECR
          │
          ▼
Create EKS Cluster
          │
          ▼
Create Fargate Profile
          │
          ▼
kubectl apply
          │
          ▼
Pods Scheduled to Fargate
          │
          ▼
Application Running
```

---

# Step 1: Build the Docker Image

Build your application image.

```bash
docker build -t my-webapp .
```

Verify:

```bash
docker images
```

---

# Step 2: Create an Amazon ECR Repository

```bash
aws ecr create-repository \
    --repository-name my-webapp
```

---

# Step 3: Tag the Image

```bash
docker tag my-webapp:latest \
123456789012.dkr.ecr.us-east-1.amazonaws.com/my-webapp:latest
```

---

# Step 4: Login to Amazon ECR

```bash
aws ecr get-login-password \
| docker login \
--username AWS \
--password-stdin \
123456789012.dkr.ecr.us-east-1.amazonaws.com
```

---

# Step 5: Push the Image

```bash
docker push \
123456789012.dkr.ecr.us-east-1.amazonaws.com/my-webapp:latest
```

Now your application image is stored in Amazon ECR.

---

# Step 6: Create an EKS Cluster

Create an EKS cluster **without worker nodes**.

Example using `eksctl`:

```bash
eksctl create cluster \
  --name demo-cluster \
  --region us-east-1 \
  --fargate
```

This command creates:

- VPC
- Subnets
- Security Groups
- IAM Roles
- EKS Control Plane
- Default Fargate Profile

No EC2 instances are created.

---

# Step 7: Create a Fargate Profile

A **Fargate Profile** tells EKS **which Pods should run on Fargate**.

Example:

```bash
eksctl create fargateprofile \
    --cluster demo-cluster \
    --name fp-default \
    --namespace default
```

You can create profiles for multiple namespaces.

Example:

```
default
production
monitoring
```

Pods created in these namespaces will automatically run on Fargate.

---

# Step 8: Create the Deployment

`deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: webapp

spec:
  replicas: 2

  selector:
    matchLabels:
      app: webapp

  template:

    metadata:
      labels:
        app: webapp

    spec:
      containers:
      - name: webapp
        image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-webapp:latest

        ports:
        - containerPort: 80
```

---

# Step 9: Create the Service

`service.yaml`

```yaml
apiVersion: v1
kind: Service

metadata:
  name: webapp-service

spec:
  selector:
    app: webapp

  ports:
  - port: 80
    targetPort: 80

  type: LoadBalancer
```

On AWS, a `LoadBalancer` Service provisions an **Elastic Load Balancer (ELB)** automatically.

---

# Step 10: Deploy the Application

Apply the manifests:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

---

# Step 11: What Happens Internally?

When you apply the Deployment:

1. Kubernetes receives the Deployment.
2. The Deployment creates the requested Pods.
3. The Kubernetes Scheduler checks the Fargate Profiles.
4. Since the Pods match the configured namespace, they are scheduled to Fargate.
5. AWS provisions the required compute resources (without creating visible EC2 instances).
6. Fargate pulls the Docker image from Amazon ECR.
7. The containers start running.
8. The Service exposes the Pods.
9. AWS creates an Elastic Load Balancer for external access (if using `LoadBalancer`).

---

# Internal Flow

```
kubectl apply
       │
       ▼
EKS API Server
       │
       ▼
Deployment
       │
       ▼
Pods Created
       │
       ▼
Scheduler
       │
       ▼
Matches Fargate Profile?
       │
       ▼
AWS Fargate
       │
       ▼
Provision Compute
       │
       ▼
Pull Image from Amazon ECR
       │
       ▼
Run Container
       │
       ▼
LoadBalancer Service
       │
       ▼
Users Access Application
```

---

# Verify the Deployment

List Pods:

```bash
kubectl get pods
```

List Services:

```bash
kubectl get services
```

Describe a Pod:

```bash
kubectl describe pod <pod-name>
```

---

# Scaling the Application

Increase the number of replicas:

```bash
kubectl scale deployment webapp --replicas=5
```

Fargate automatically provisions additional compute for the new Pods.

---

# Updating the Application

Build and push a new image:

```bash
docker build -t my-webapp:v2 .
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-webapp:v2
```

Update the Deployment:

```bash
kubectl set image deployment/webapp \
webapp=123456789012.dkr.ecr.us-east-1.amazonaws.com/my-webapp:v2
```

Kubernetes performs a rolling update while Fargate launches new Pods with the updated image.

---

# Advantages of EKS with Fargate

- No EC2 instance management.
- No node patching or maintenance.
- Automatic compute provisioning.
- Pay only for the CPU and memory used by running Pods.
- Strong isolation because each Pod runs in its own Fargate environment.
- Ideal for microservices, APIs, and event-driven workloads.

---

# Limitations

- Higher cost for always-on, large-scale workloads compared to EC2.
- Supports only Linux containers.
- DaemonSets are not supported on Fargate.
- Some networking and storage features have limitations compared to EC2-backed nodes.
- Less control over the underlying infrastructure.

---

# Complete End-to-End Flow

```
Developer
     │
     ▼
Write Application
     │
     ▼
Docker Build
     │
     ▼
Amazon ECR
     │
     ▼
kubectl apply
     │
     ▼
Amazon EKS
     │
     ▼
Fargate Profile
     │
     ▼
AWS Fargate
     │
     ▼
Container Starts
     │
     ▼
Elastic Load Balancer
     │
     ▼
End Users
```

---

# Summary

Deploying applications on **Amazon EKS with Fargate** simplifies Kubernetes operations by eliminating the need to manage worker nodes. The typical process is:

1. Build the Docker image.
2. Push the image to Amazon ECR.
3. Create an EKS cluster with Fargate support.
4. Create a Fargate Profile for the target namespace.
5. Deploy the application using Kubernetes manifests.
6. Fargate automatically provisions compute resources, pulls the image from ECR, and runs the Pods.
7. Expose the application using a Kubernetes Service (typically `LoadBalancer`) for external access.

This approach allows you to focus entirely on your applications while AWS manages the underlying infrastructure.