## What is EKS?
    - Managed Kubernetes Service
    - AWS manages Master Nodes for us
    - Install all the necessary apps pre-installed on Master nodes like **Container Runtime**, **Master Processes** by AWS
    - Takes care of Scaling and backups
    - We can focus on deploying our applications
    - We create only Worker Nodes

##  How to use EKS?
1. Setup 
    - Create a VPC in AWS
    - Create an IAM role with Security Group
    - IAM role -> Create AWS user
    - Security Group -> list of permissions
2. Create Cluster Control Plane (through aws ui or cli -preferred)
    - Choose cluster name, k8s version
    - Choose region and VPC for your cluster
    - Set security for your cluster
3. Create Worker Nodes and connect to cluster
    - Create as a Node Group (group of nodes)
    - Choose cluster it will attach to
    - Define Security Group, select instance type, resources
    - Define max and min number of Nodes (Autoscaling)
4. Connect thought local Machine (to deploy applications)
    - Use kubectl to talk to remote cluster

To simplify this steps we can use **eksctl** for Aws EKS and it is not provided by aws but from opensource community

eksctl will remove all the overhead like creating public and private subnets and other essential configs with the stater defaults.


# Create Kubernetes cluster on AWS EKS using eksctl
1. Install eksctl for your OS
2. eksctl needs to authenticate with AWS (~/.aws/credentials aws_acces_key_id and aws_secret_access_key)
3. To create cluster (here node means worker)
    $ eksctl create cluster \
    > --name test-cluster \
    > --version 1.17\
    > --region eu-central-1 \
    > --nodegroup-name test-linux-nodes \     
    > --node-type t2.micro \
    > --nodes 2

kubeconfig as "/Users/subbu/.kube/config 
by running step 3 it will generate a config  in the above spicified location which contains about the cluster so that Kubectl on our local machine can connect to the remote cluster. it contains:
1. enpoint of the cluster
2. certificates and some other required details

So now when you use Kubectl to connect with the cluster this file will will help to locate remote cluster

$ kubectl get nodes
$ kubectl get nodes
$ eksctl delete cluster --name test-cluster (will delete all the culster related things)





