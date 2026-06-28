# Amazon EKS (Elastic Kubernetes Service)

## What is EKS?

Amazon **Elastic Kubernetes Service (EKS)** is a **fully managed Kubernetes service** provided by AWS. It removes the complexity of managing the Kubernetes control plane, allowing you to focus on deploying and managing your applications.

### Key Features

- Fully managed Kubernetes service by AWS
- AWS manages the **Control Plane (Master Nodes)** for you
- AWS installs and maintains all Kubernetes master components such as:
  - API Server
  - etcd
  - Scheduler
  - Controller Manager
  - Container Runtime (where applicable)
- Automatic control plane scaling
- High availability across multiple Availability Zones
- Automatic backups and updates of the control plane
- Security patches managed by AWS
- You only need to manage the **Worker Nodes** (or even those can be managed using EKS Managed Node Groups)

---

# EKS Architecture

```
                     AWS EKS

          +---------------------------+
          |      Control Plane        |
          |---------------------------|
          | Kubernetes API Server     |
          | etcd                      |
          | Scheduler                 |
          | Controller Manager        |
          +---------------------------+
                    (Managed by AWS)

                           │
                           │
                Kubernetes API Communication
                           │

       +----------------------------------------+
       |           Worker Node Group            |
       |----------------------------------------|
       | EC2 Instance                           |
       | Kubelet                               |
       | kube-proxy                            |
       | Container Runtime                     |
       | Your Pods                             |
       +----------------------------------------+

```

---

# Manual Steps to Create an EKS Cluster

If creating an EKS cluster manually (using the AWS Console or AWS CLI), the general workflow is as follows.

## Step 1: Setup Networking and Permissions

### Create a VPC

Create a Virtual Private Cloud (VPC) that will host your Kubernetes cluster.

The VPC typically contains:

- Public Subnets
- Private Subnets
- Internet Gateway
- NAT Gateway
- Route Tables

### Create IAM Roles

Create the required IAM roles for:

- EKS Control Plane
- Worker Nodes

These IAM roles provide permissions for AWS resources.

### Configure Security Groups

Security Groups act as virtual firewalls.

They define:

- Allowed inbound traffic
- Allowed outbound traffic
- Communication between worker nodes and the control plane

---

## Step 2: Create the EKS Control Plane

You can create the control plane using:

- AWS Console
- AWS CLI (recommended)
- eksctl

During creation, specify:

- Cluster Name
- Kubernetes Version
- AWS Region
- VPC
- Security Groups
- IAM Role

AWS then provisions the Kubernetes control plane.

---

## Step 3: Create Worker Nodes

Worker Nodes are EC2 instances that actually run your containers.

Typically, Worker Nodes are created as a **Node Group**.

While creating the Node Group, specify:

- Cluster to attach
- Instance Type
- Disk Size
- Number of Nodes
- Auto Scaling configuration
  - Minimum Nodes
  - Desired Nodes
  - Maximum Nodes

Example:

```
Cluster
   │
   ├── Node Group
   │      ├── EC2 Worker Node
   │      ├── EC2 Worker Node
   │      └── EC2 Worker Node
```

---

## Step 4: Connect from Your Local Machine

Install:

- AWS CLI
- kubectl

Configure your kubeconfig so kubectl knows how to connect to the remote cluster.

Once configured:

```bash
kubectl get nodes
kubectl get pods
kubectl get namespaces
```

---

# Using eksctl (Recommended)

Creating all AWS networking resources manually is time-consuming.

The **eksctl** tool automates almost the entire EKS provisioning process.

> **Note:** `eksctl` is **not an official AWS CLI tool**. It is an open-source project built specifically for simplifying EKS cluster creation.

---

## What does eksctl create automatically?

By default, `eksctl` creates:

- VPC
- Public Subnets
- Private Subnets
- Internet Gateway
- Route Tables
- Security Groups
- IAM Roles
- CloudFormation Stacks
- EKS Control Plane
- Managed Node Group
- Auto Scaling Group

You can create an entire production-ready Kubernetes cluster with a single command.

---

# eksctl Architecture

```
eksctl
   │
   │
   ├── Creates VPC
   ├── Creates Public Subnets
   ├── Creates Private Subnets
   ├── Creates Internet Gateway
   ├── Creates Route Tables
   ├── Creates Security Groups
   ├── Creates IAM Roles
   ├── Creates EKS Cluster
   └── Creates Worker Nodes
```

---

# Installing eksctl

Install `eksctl` based on your operating system.

Documentation:

https://eksctl.io/

---

# AWS Authentication

Before creating a cluster, configure AWS credentials.

AWS CLI stores credentials in:

```
~/.aws/credentials
```

Example:

```ini
[default]
aws_access_key_id=YOUR_ACCESS_KEY
aws_secret_access_key=YOUR_SECRET_KEY
region=eu-central-1
```

Verify authentication:

```bash
aws sts get-caller-identity
```

---

# Creating an EKS Cluster

Example:

```bash
eksctl create cluster \
  --name test-cluster \
  --version 1.17 \
  --region eu-central-1 \
  --nodegroup-name test-linux-nodes \
  --node-type t2.micro \
  --nodes 2
```

### Parameters

| Parameter | Description |
|-----------|-------------|
| `--name` | Cluster Name |
| `--version` | Kubernetes Version |
| `--region` | AWS Region |
| `--nodegroup-name` | Worker Node Group Name |
| `--node-type` | EC2 Instance Type |
| `--nodes` | Number of Worker Nodes |

---

# What Happens Internally?

Running the above command performs the following:

1. Creates a VPC (if not provided)
2. Creates Public & Private Subnets
3. Creates Internet Gateway
4. Creates Route Tables
5. Creates IAM Roles
6. Creates Security Groups
7. Creates the EKS Control Plane
8. Creates Worker Nodes
9. Configures Auto Scaling
10. Generates kubeconfig for your local machine

---

# kubeconfig

After the cluster is created, `eksctl` automatically generates a Kubernetes configuration file.

Location:

```
~/.kube/config
```

Example:

```
/Users/subbu/.kube/config
```

This file allows `kubectl` to communicate with the remote EKS cluster.

The kubeconfig contains:

- Cluster Endpoint
- Cluster Certificate
- Authentication Information
- User Credentials
- Cluster Name
- Context Information

Whenever you execute:

```bash
kubectl get nodes
```

kubectl reads the kubeconfig file to determine:

- Which cluster to connect to
- How to authenticate
- Which certificates to use

---

# Verify the Cluster

Check worker nodes:

```bash
kubectl get nodes
```

Check all pods:

```bash
kubectl get pods --all-namespaces
```

Check namespaces:

```bash
kubectl get namespaces
```

---

# Delete the Cluster

To delete the entire cluster and all associated AWS resources created by `eksctl`:

```bash
eksctl delete cluster --name test-cluster
```

This deletes:

- EKS Cluster
- Worker Nodes
- Auto Scaling Group
- CloudFormation Stack
- Security Groups
- IAM Resources (created by eksctl)
- VPC (if created by eksctl)

---

# Can eksctl Create a VPC?

**Yes.**

By default, `eksctl` automatically creates a new VPC if one is not provided.

The automatically created VPC includes:

- Public Subnets
- Private Subnets
- Internet Gateway
- Route Tables
- Security Groups

This is why most tutorials only require a single `eksctl create cluster` command.

---

# Using an Existing VPC

If your organization already has a VPC, you can instruct `eksctl` to use it instead of creating a new one.

Example:

```bash
eksctl create cluster \
    --name production-cluster \
    --region eu-central-1 \
    --vpc-public-subnets subnet-11111,subnet-22222 \
    --vpc-private-subnets subnet-33333,subnet-44444
```

In this case:

- No new VPC is created.
- The EKS cluster is deployed into the existing VPC.

---

# Manual vs eksctl

| Manual Setup | Using eksctl |
|--------------|--------------|
| Create VPC | Automatically creates VPC |
| Create Subnets | Automatically creates Subnets |
| Configure Route Tables | Automatically configures |
| Create Security Groups | Automatically creates |
| Create IAM Roles | Automatically creates |
| Create Control Plane | Automatically creates |
| Create Worker Nodes | Automatically creates |
| Configure kubeconfig | Automatically configures |

---

# Summary

- Amazon EKS is a managed Kubernetes service.
- AWS manages the Kubernetes control plane.
- You primarily manage the worker nodes (or use managed node groups).
- `eksctl` is the easiest way to provision an EKS cluster.
- By default, `eksctl` automatically creates networking resources such as the VPC, subnets, IAM roles, and security groups.
- After cluster creation, `kubectl` uses the generated `~/.kube/config` file to communicate with the EKS cluster.
- The entire cluster and its associated resources can be deleted with a single `eksctl delete cluster` command.


# Kubernetes Namespaces

## What is a Namespace?

A **Namespace** is a logical partition inside a Kubernetes cluster that allows you to organize, isolate, and manage resources.

Think of a namespace as a **virtual cluster** within a physical Kubernetes cluster. It helps multiple teams, applications, or environments share the same cluster without interfering with each other.

> **Note:** Namespaces provide **logical isolation**, not physical isolation. All namespaces share the same Kubernetes control plane and worker nodes.

---

# Why Do We Need Namespaces?

Imagine your organization has a single Kubernetes cluster used by multiple teams.

For example:

- Team A → E-commerce Application
- Team B → Payment Service
- Team C → Monitoring & Logging

Without namespaces, all Kubernetes resources would exist together in one place.

```
Kubernetes Cluster

├── Pod
├── Deployment
├── Service
├── ConfigMap
├── Secret
├── Pod
├── Deployment
└── Service
```

As the cluster grows, it becomes difficult to:

- Identify which resources belong to which application
- Prevent naming conflicts
- Apply different permissions
- Control resource usage
- Manage different environments

Namespaces solve all of these problems.

---

# How Namespaces Work

A Kubernetes cluster can contain multiple namespaces.

```
Kubernetes Cluster

├── Namespace: development
│      ├── Pods
│      ├── Services
│      ├── Deployments
│      └── ConfigMaps
│
├── Namespace: testing
│      ├── Pods
│      ├── Services
│      ├── Deployments
│      └── Secrets
│
└── Namespace: production
       ├── Pods
       ├── Services
       ├── Deployments
       └── ConfigMaps
```

Each namespace maintains its own set of Kubernetes resources.

---

# Preventing Resource Name Conflicts

Resource names must be unique **within a namespace**, but the same name can be reused in different namespaces.

Without namespaces:

```
Deployment Name: backend

❌ Error:
Deployment "backend" already exists
```

With namespaces:

```
development
│
└── backend

production
│
└── backend
```

Both deployments can exist because they belong to different namespaces.

---

# Organizing Applications

Namespaces help organize related resources.

Example:

```
Namespace: ecommerce

├── frontend
├── backend
├── payment
├── cart
├── mysql
└── redis
```

Another application:

```
Namespace: inventory

├── api
├── database
├── redis
└── worker
```

Resources remain grouped and easy to manage.

---

# Separating Environments

Instead of creating separate Kubernetes clusters for every environment, many organizations use namespaces.

Example:

```
Kubernetes Cluster

├── dev
├── test
└── prod
```

Each environment has its own:

- Pods
- Services
- Deployments
- ConfigMaps
- Secrets

This approach reduces infrastructure costs while maintaining logical separation.

---

# Resource Isolation

Namespaces isolate Kubernetes resources from one another.

Resources that belong to one namespace are independent of resources in another namespace.

For example:

```
Namespace: development

Pods:
- frontend
- backend

Namespace: production

Pods:
- frontend
- backend
```

Although the pod names are identical, Kubernetes treats them as different resources because they belong to different namespaces.

---

# Resource Quotas

Namespaces allow administrators to limit resource consumption.

Example:

| Namespace | CPU Limit | Memory Limit |
|-----------|----------:|-------------:|
| development | 4 CPU | 8 GB |
| testing | 2 CPU | 4 GB |
| production | 20 CPU | 64 GB |

This prevents one namespace from consuming all cluster resources.

---

# Access Control (RBAC)

Namespaces work with Kubernetes Role-Based Access Control (RBAC).

Example permissions:

| User | Namespace Access |
|------|------------------|
| Developers | development |
| QA Team | testing |
| DevOps Team | production |
| Cluster Admin | All Namespaces |

This improves security by restricting users to only the namespaces they need.

---

# Built-in Namespaces

Every Kubernetes cluster includes several default namespaces.

## default

The default namespace used when no namespace is specified.

Example:

```bash
kubectl get pods
```

This command retrieves pods from the `default` namespace.

---

## kube-system

Contains Kubernetes system components.

Examples:

- CoreDNS
- kube-proxy
- Metrics Server
- Networking Plugins

Example:

```bash
kubectl get pods -n kube-system
```

---

## kube-public

Contains publicly readable resources.

Generally used for information that should be accessible throughout the cluster.

---

## kube-node-lease

Stores heartbeat information for worker nodes.

Helps Kubernetes efficiently detect node failures.

---

# Viewing Namespaces

List all namespaces:

```bash
kubectl get namespaces
```

or

```bash
kubectl get ns
```

Example output:

```
NAME              STATUS   AGE
default           Active   12d
kube-system       Active   12d
kube-public       Active   12d
kube-node-lease   Active   12d
development       Active    5d
production        Active    5d
```

---

# Creating a Namespace

Create a namespace:

```bash
kubectl create namespace development
```

Verify:

```bash
kubectl get namespaces
```

---

# Deploying Resources into a Namespace

Create a deployment inside a namespace:

```bash
kubectl create deployment nginx \
    --image=nginx \
    --namespace=development
```

or

```bash
kubectl create deployment nginx \
    --image=nginx \
    -n development
```

---

# Listing Resources from a Namespace

List pods:

```bash
kubectl get pods -n development
```

List services:

```bash
kubectl get services -n development
```

List deployments:

```bash
kubectl get deployments -n development
```

---

# Switching Namespace Permanently

Instead of specifying `-n` every time, change the current namespace.

```bash
kubectl config set-context --current --namespace=development
```

Now simply run:

```bash
kubectl get pods
```

It automatically uses the `development` namespace.

---

# Deleting a Namespace

Delete a namespace:

```bash
kubectl delete namespace development
```

Deleting a namespace removes **all resources** inside it, including:

- Pods
- Deployments
- Services
- ConfigMaps
- Secrets
- Jobs
- StatefulSets

---

# When Should You Use Separate Clusters?

Namespaces are suitable for logical separation, but sometimes separate clusters are required.

Use different clusters when you need:

- Complete security isolation
- Different Kubernetes versions
- Different AWS accounts
- Different cloud regions
- Separate billing
- Disaster recovery isolation

---

# Cluster vs Namespace

| Kubernetes Cluster | Namespace |
|--------------------|-----------|
| Physical Kubernetes infrastructure | Logical partition inside a cluster |
| Has its own Control Plane | Shares the cluster's Control Plane |
| Has its own Worker Nodes | Shares the same Worker Nodes |
| Complete isolation | Logical isolation |
| More expensive | Cost-effective |
| Used for strong isolation | Used for organizing workloads |

---

# Summary

- A Namespace is a logical partition inside a Kubernetes cluster.
- It helps organize applications, teams, and environments.
- Namespaces prevent resource name conflicts.
- They support access control using RBAC.
- Resource quotas can be applied per namespace.
- Multiple environments (development, testing, production) can coexist within the same cluster.
- Namespaces provide logical isolation, while all namespaces still share the same control plane and worker nodes.
- Kubernetes includes four built-in namespaces: `default`, `kube-system`, `kube-public`, and `kube-node-lease`.

Namespaces are one of the most fundamental Kubernetes concepts and are widely used in production environments to efficiently manage shared clusters.



# Deploying a Web Application to a Kubernetes Cluster

## Overview

After creating a Kubernetes cluster, the next step is to deploy your application.

A typical deployment involves:

1. Building a Docker image of your application
2. Pushing the image to a container registry
3. Creating Kubernetes Deployment and Service manifests
4. Applying the manifests to the cluster
5. Accessing the application

---

# Deployment Workflow

```
Developer
     │
     │
     ▼
Build Docker Image
     │
     ▼
Push Image to Docker Hub / ECR
     │
     ▼
kubectl apply
     │
     ▼
Kubernetes Deployment
     │
     ▼
Pods Created
     │
     ▼
Service Exposes Pods
     │
     ▼
Application Accessible
```

---

# Step 1: Build the Docker Image

Assume your web application already contains a `Dockerfile`.

Build the Docker image:

```bash
docker build -t my-webapp:v1 .
```

Verify the image:

```bash
docker images
```

Example:

```
REPOSITORY     TAG
my-webapp      v1
```

---

# Step 2: Push the Image to a Container Registry

Kubernetes nodes need to pull your Docker image from a registry.

Common registries include:

- Docker Hub
- Amazon ECR (Elastic Container Registry)
- Google Artifact Registry
- Azure Container Registry

### Tag the image

Example (Docker Hub):

```bash
docker tag my-webapp:v1 username/my-webapp:v1
```

Push the image:

```bash
docker push username/my-webapp:v1
```

---

# Step 3: Create a Deployment

A Deployment manages Pods and ensures the desired number of replicas are running.

Create a file named **deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: webapp-deployment

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

        image: username/my-webapp:v1

        ports:
        - containerPort: 80
```

### Explanation

- **replicas** → Number of Pod copies
- **selector** → Identifies Pods managed by the Deployment
- **template** → Pod template
- **image** → Docker image to deploy
- **containerPort** → Port exposed by the container

---

# Step 4: Create a Service

Pods receive temporary IP addresses.

A Service provides a stable endpoint for accessing the Pods.

Create **service.yaml**

```yaml
apiVersion: v1
kind: Service

metadata:
  name: webapp-service

spec:
  selector:
    app: webapp

  ports:
  - protocol: TCP
    port: 80
    targetPort: 80

  type: LoadBalancer
```

### Service Types

| Type | Description |
|------|-------------|
| ClusterIP | Accessible only within the cluster (default) |
| NodePort | Accessible using `<NodeIP>:Port` |
| LoadBalancer | Creates an external cloud load balancer |
| ExternalName | Maps to an external DNS name |

For AWS EKS, `LoadBalancer` automatically provisions an AWS Elastic Load Balancer.

---

# Step 5: Deploy the Application

Apply the Deployment:

```bash
kubectl apply -f deployment.yaml
```

Apply the Service:

```bash
kubectl apply -f service.yaml
```

---

# Step 6: Verify the Deployment

Check Deployments:

```bash
kubectl get deployments
```

Example:

```
NAME                 READY   UP-TO-DATE   AVAILABLE
webapp-deployment    2/2     2            2
```

---

Check Pods:

```bash
kubectl get pods
```

Example:

```
NAME                                   READY
webapp-deployment-6fd9cbf5c8-abc12      1/1
webapp-deployment-6fd9cbf5c8-def45      1/1
```

---

Check Services:

```bash
kubectl get services
```

Example:

```
NAME              TYPE           CLUSTER-IP      EXTERNAL-IP
webapp-service    LoadBalancer   10.100.0.45     a1b2c3.amazonaws.com
```

---

# Step 7: Access the Application

If using a **LoadBalancer** service:

```
http://EXTERNAL-IP
```

Example:

```
http://a1b2c3.amazonaws.com
```

For EKS, AWS automatically creates an Elastic Load Balancer and assigns a public DNS name.

---

# Complete Resource Relationship


Deployment
     │
     │ creates
     ▼
Pods (2 replicas)
     │
     │ selected by label
     ▼
Service
     │
     │
Load Balancer
     │
     ▼
Users
```

---

# Scaling the Application

Increase replicas:

```bash
kubectl scale deployment webapp-deployment --replicas=5
```

Verify:

```bash
kubectl get pods
```

Now Kubernetes creates 5 Pods.

---

# Updating the Application

Build and push a new image:

```bash
docker build -t username/my-webapp:v2 .
docker push username/my-webapp:v2
```

Update the Deployment:

```bash
kubectl set image deployment/webapp-deployment \
webapp=username/my-webapp:v2
```

Kubernetes performs a **Rolling Update**, replacing Pods one by one without downtime.

---

# Check Rollout Status

```bash
kubectl rollout status deployment/webapp-deployment
```

View rollout history:

```bash
kubectl rollout history deployment/webapp-deployment
```

---

# Roll Back to the Previous Version

```bash
kubectl rollout undo deployment/webapp-deployment
```

---

# Delete the Application

Delete the Deployment:

```bash
kubectl delete deployment webapp-deployment
```

Delete the Service:

```bash
kubectl delete service webapp-service
```

Or delete both manifests:

```bash
kubectl delete -f deployment.yaml
kubectl delete -f service.yaml
```

---

# Best Practices

- Use versioned Docker image tags (avoid `latest` in production).
- Store sensitive information in **Secrets**.
- Store configuration in **ConfigMaps**.
- Set CPU and memory requests/limits.
- Use readiness and liveness probes.
- Run multiple replicas for high availability.
- Use namespaces to organize applications.
- Prefer `LoadBalancer` or an Ingress Controller for exposing web applications in cloud environments.

---

# Summary

Deploying a web application to Kubernetes typically follows these steps:

1. Build the Docker image.
2. Push the image to a container registry.
3. Create a Deployment manifest.
4. Create a Service manifest.
5. Apply the manifests using `kubectl`.
6. Verify that Pods and Services are running.
7. Access the application through the Service or Load Balancer.
8. Scale, update, or roll back the application as needed.

This declarative approach allows Kubernetes to automatically maintain the desired state of your application, ensuring reliability, scalability, and ease of management.


# Storing Docker Images in AWS (Amazon ECR)

## Overview

When deploying applications to **Amazon EKS (Elastic Kubernetes Service)**, your Docker images need to be stored in a **Container Registry** so that Kubernetes worker nodes can pull them when creating Pods.

The recommended registry for AWS is **Amazon Elastic Container Registry (ECR)**.

---

# What is Amazon ECR?

**Amazon Elastic Container Registry (ECR)** is a fully managed Docker container registry provided by AWS.

It allows you to:

- Store Docker images securely
- Version container images
- Integrate with AWS IAM for authentication
- Automatically scan images for vulnerabilities
- Easily deploy images to Amazon EKS, ECS, or EC2

---

# Why Use Amazon ECR?

- Fully managed by AWS
- Seamless integration with Amazon EKS
- Secure authentication using IAM
- Highly available and scalable
- Supports image vulnerability scanning
- Supports lifecycle policies to remove old images
- No Docker Hub pull rate limits

---

# Docker Image Deployment Workflow

```
Application Source Code
          │
          ▼
docker build
          │
          ▼
Docker Image
          │
          ▼
Amazon ECR Repository
          │
          ▼
kubectl apply
          │
          ▼
Kubernetes Deployment
          │
          ▼
Worker Nodes Pull Image
          │
          ▼
Running Pods
```

---

# Step 1: Create an ECR Repository

Create a new repository using the AWS CLI.

```bash
aws ecr create-repository \
    --repository-name my-webapp
```

Example output:

```
Repository URI:

123456789012.dkr.ecr.us-east-1.amazonaws.com/my-webapp
```

---

# Step 2: Build the Docker Image

Build your application image.

```bash
docker build -t my-webapp .
```

Verify the image:

```bash
docker images
```

Example:

```
REPOSITORY     TAG
my-webapp      latest
```

---

# Step 3: Tag the Image

Tag the image using the ECR repository URI.

```bash
docker tag my-webapp:latest \
123456789012.dkr.ecr.us-east-1.amazonaws.com/my-webapp:latest
```

---

# Step 4: Authenticate Docker with ECR

Before pushing the image, authenticate Docker with Amazon ECR.

```bash
aws ecr get-login-password \
| docker login \
--username AWS \
--password-stdin \
123456789012.dkr.ecr.us-east-1.amazonaws.com
```

If successful:

```
Login Succeeded
```

---

# Step 5: Push the Image

Push the Docker image to Amazon ECR.

```bash
docker push \
123456789012.dkr.ecr.us-east-1.amazonaws.com/my-webapp:latest
```

Example output:

```
latest: digest: sha256:...
```

Your image is now stored in Amazon ECR.

---

# Step 6: Use the Image in Kubernetes

Update your `deployment.yaml` to reference the image stored in ECR.

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

Deploy the application:

```bash
kubectl apply -f deployment.yaml
```

The EKS worker nodes will automatically pull the image from Amazon ECR.

---

# Complete Image Deployment Flow

```
Developer
     │
     ▼
Docker Build
     │
     ▼
Docker Image
     │
     ▼
Amazon ECR
     │
     ▼
Kubernetes Deployment
     │
     ▼
Worker Node Pulls Image
     │
     ▼
Pod Starts Running
```

---

# Other Container Registries

Although Amazon ECR is recommended for AWS, Kubernetes can also pull images from other registries.

| Registry | Best For |
|----------|----------|
| Amazon ECR | AWS (EKS, ECS, EC2) ⭐ |
| Docker Hub | Public images and personal projects |
| GitHub Container Registry (GHCR) | GitHub-based projects |
| Google Artifact Registry | Google Cloud |
| Azure Container Registry | Microsoft Azure |

---

# Best Practices

- Use versioned image tags (e.g., `v1.0.0`) instead of `latest` in production.
- Enable image scanning to detect vulnerabilities.
- Configure lifecycle policies to remove unused images.
- Grant only the required IAM permissions for pushing and pulling images.
- Store production images in private repositories.

---

# Summary

- Amazon ECR is AWS's fully managed Docker container registry.
- Docker images should be stored in ECR when deploying applications to Amazon EKS.
- The deployment workflow is:
  1. Build the Docker image.
  2. Create an ECR repository.
  3. Tag the image with the ECR repository URI.
  4. Authenticate Docker with ECR.
  5. Push the image to ECR.
  6. Reference the ECR image in the Kubernetes Deployment.
  7. Deploy the application using `kubectl apply`.

Amazon ECR provides a secure, scalable, and fully managed solution for storing and serving container images to Kubernetes clusters running on AWS.

