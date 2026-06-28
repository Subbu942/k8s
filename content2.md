## What is Kubernetes
- Open source container orchestration tool
- Developed by Google
- Helps manage containerized applications in different deployment environments

## What problems does Kubernetes solve? 
- Trend from Monolith to Microservices
- Increased usage of containers
- Demand for a proper way of managing those hundreds of containers


## What are the `tasks` of an orchestration tool?
- High Availability or no downtime
- Scalability or high performance
- Disaster recovery -backup and restore


## Kubernetes Architecture
- Contains at least one master node(also refered as controller) and couple of worker nodes(worker nodes are mostly refered as nodes)
- On Worker Nodes your applications are running
- On Master Node importent K8s processes are running.

Those are:
1. **API server** (Entry point to K8s cluster)
2. **Controller Manager** (Keeps track of whats happening in the cluster i.e,. whether something needs to be repaired, if container died and it needs to be restarted, etc,.)
3. **Scheduler** (Ensures Pods placement. Scheduler decides on which Node new Pod should be scheduled. Intelligent process that decides on which worker node the next container should be scheduled base on the analysis of avalible server(eg: EC2-configuration and storage availability) )
4. **etcd** Kubernetes backing store (basically holds at any time the current state of the kubernetes cluster so it has all the configuration data inside and all the status data of each node and each contatiner inside of that node. backup is make through the etcd snapshots)

5. **Virtual Network** (worker nodes and master notes talk to each other is through virtual network that spans all the nodes that are part of the cluster). Virtual network creates one unified machine

Worker Nodes -> higher workload -> much bigger and more resources than master nodes

Control Plane Nodes (master nodes) -> handful of master processes -> less resources than worker nodes

** in production environement we have at least 2 master nodes because if we loos master node we can't use worker nodes we will loose the track of worknodes


## Main Kubernetes Components
1. Pod
2. IP
3. Service
4. Ingress
5. ConfigMap
6. Secret
7. Volume
8. Deployment 
9. StatefulSet
10. DaemonSet

- **Pod** : Smallest unit in Kubernetes, Abstraction over container (a layer on top of container). Kubernetes want to abstract a way the container runtime or container technologies so that we can replace them if we want to and also because we don't have to dircectly work with docker or whatever container technology we use in a K8s. We only interact with the k8s layer so we have an application pod which is our own application and that will maybe use a database pod with its own container. Pod is usally ment to run 1 Application.But you can also run multiple containers inside one pod but usually it's only the case if we have one main application container and the helper container or some side service that has to run inside of that pod

- **IP Address** : Each Pod get its own IP Address, and each pod can communicate with each other using that ip address which is an internal ip address not the public one.
When a Pod dies on re-creation it will get new IP address so it is not stable to use for communication.

- **Service** : Service solves the problem of IP makeing it stable. Service is a static/permanent ip address that can be attached to each pod. Lifecycle of Pod and Service are not connected even if the POd dies the service won't when the Pod re-create it will be attached to the same service. Service is also a `load balancer` multiple worker nodes running clone of the same taks in different pods in different worker nodes will be managed by the same service(static ip address)

- **Ingress** : Services are for the internal communications between the pods but Ingress is created at Node level to communicate with the external world

- **ConfigMap**: External Configuration of our application like database url, etc,. In K8s ConfigMap is connected to pod(s). ConfigMap is for non-configential data only

- **Secret** : Used to store secret data but values should be stored in Base64 encrypted formate. Use it as environment variables or as a properties file

- **Volume**: Database data is store persistent in the volumes so that even if it restarts the data will be accessable. Volume can be on local machine(same worker node) or remote, outside of the K8s cluster
** K8s doesn't manage data persistance we have to manage them on our own

- **Deployment**: Blueprint of "application" Pods. It contains all the required things to start a new pod with the same application in different worker nodes (app_deployment.yaml). We create Deployemnts not the Pods. Deployemnt is an Abstration layer on top of Pods.

- **StatefulSet**: to avoid inconsistencies we deploy database using StatefulSet insted of Deployement

** Deployment -> for stateLess Apps
** StatefulSet -> for stateFull Apps or Databases
** Databases are often hosted outside of Kubernetes cluster (best practice too).

## Kubernetes Configuration
The request accepted by the Kubernetes `API SERVER` is either YAML or json. as we know API Server in the master node (control plane) is the entrypoint to any request either from UI,API or CLI

In K8s Configuration is `Declarative` (i.e,. yaml file)

Example:

```yaml
//nginx-deployement.yaml
apiVersion: apps/v1
kind: Deployment
metadata: 
    name: nginx-deployemnt
    labels: ...
spec:
    replicas: 2
    selector: ...
    template: ...
```
```yaml
//nginx-service.yaml
apiVersion: v1
kind:Service
metadata:
    name: nginx-service
spec:
    selector: ...
    ports: ...
```    

- Each Configuration File has 3 parts
1. metadata
2. specification
3. status (Automatically generated and added by Kubernetes)
- Kubernetes always compares what is Desired state and what is Actual state. if they does'nt match K8s will fix it
till the desired = Actual

** Frome Where does K8s get this status data? From etcd


## YAML Configuration Files
1. YAML is Human Friendly data serialization standard for all programming languages
2. systax: strict indentation!
3. store the cofig file with our code (application code)


## MiniKube
- Master and Node processes run on ONE machine(one Node) in Local system. on top of Docker container pre-instelled 


- What is Kubectl? Command line tool for K8s cluster

Kubectl -> Master Process -> API server -> enables interaction with cluster

Kubectl -> Worker processes -> enable pods to run on node (create pods, create services, destroy pods)

Kubctel -> Minikube cluster
        -> Cloud cluster (AWS,GCS)

- MiniKube can run either as a container or Virtual Machine on our laptop
---
- Basic Requriements 
    2cpu's or more,

    2GB of free memory,

    20GB of free disk space,

    Internet connection,

    Container or virtual machine manager, such as: Docker, Hyperkit, Hyper-V, KVM, Podman, VirtualBox or VMWare
--- 

## Create and start the cluster
```cmd
$ minikube start --driver docker

$ minkube status (you started a container like any other container in docker)

**Minikube has kubectl as dependency (it is installed along with minikube)

$ kubctl get node

- Minikube CLI -> is for startup/deleting the cluster like eksctl

- Kubectl CLI -> is for configuring the Minikube cluster
```
--- 

**Follow kubernetes latest documentation for getting the configs right**

## K8s Components Overview (Create 4 K8s Config Files)
1. ConfigMap -> MongoDB Endpoint
2. Secret -> MongoDB User & Pwd
3. Deployment & Service -> MogoDB Application with internal Service
4. Deployment & Service -> Our own WebApp with external service



## Creating all the components in K8s

$ kubectl get pod
$ kubectl apply -f mongo-config.yaml
$ kubectl apply -f mongo-secret.yaml
$ kubectl apply -f mongo.yaml
$ kubectl apply -f webapp.yaml # -f is for file and order of files should be sequential to dependencies 

$ kubectl get all
$ kubectl get configmap
$ kubectl get secret
$ kubectl get pod
$ kubectl --help
$ kubectl describe service webapp-service
$ kubectl describe pod webapp-deployment-dcffd6bcc-gsmlt
$ kubectl logs webapp-deployment-dcffd6bcc-gsmlt
$ kubectl logs webapp-deployment-dcffd6bcc-gsmlt -f # for streming the logs
$ kubectl get svc
$ minikube ip # to get the minkube ip which we use for accessing application from web
$ kubectl get node -o wide # will also give you ip (internal ip) to access through web





