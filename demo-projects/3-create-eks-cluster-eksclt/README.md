## Demo Project - Create an AWS EKS Cluster with eksctl

### Topics of the Demo Project
Create EKS cluster with eksctl

### Technologies Used
- Kubernetes
- AWS EKS
- eksctl
- Linux

### Project Description
- Create EKS cluster using eksctl tool that reduces the manual effort of creating an EKS cluster

#### Steps to install eksctl
On my Mac with M2 processor I could install eksctl using homebrew:
```sh
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
```

The official [eksctl installation guide](https://github.com/weaveworks/eksctl#installation) recommends to download it from the official site, though. So we execute the following commands:
```sh
ARCH=arm64
PLATFORM=$(uname -s)_$ARCH

curl -sLO "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"

# (Optional) Verify checksum
curl -sL "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | shasum -a 256 --check
# => eksctl_Darwin_arm64.tar.gz: OK

tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz

sudo mv /tmp/eksctl /usr/local/bin
``` 

#### Steps to configure AWS credentials to connect eksctl with your AWS account
Because we already have configured credentials for awscli we can use the same configuration for eksctl. Otherwhise we would have to tell eksctl with which account and which user we want to connect. Using the file, that was downloaded when we created the access key for the admin user we would execute the following command:

```sh
aws configure
  AWS Access Key ID [None]: # enter the AWS access key id from the downloaded .csv file
  AWS Secret Access Key [None]: # enter the AWS secret access key from the downloaded .csv file
  Default region name [None]: eu-central-1 # Frankfurt (eu-west-3 for Paris)
  Default output format [None]: json
```

This configuration will be used for all subsequent eksctl (or awscli) commands. The configuration itself is stored in `~/.aws/config` and `~/.aws/credentials`.

#### Steps to create an EKS cluster
 Execute the following command:
```sh
eksctl create cluster \
  --name demo-cluster \
  --version 1.26 \
  --region eu-central-1 \
  --nodegroup-name demo-nodes \
  --node-type t2.micro \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3
# =>
# 2023-05-18 15:36:18 [ℹ]  eksctl version 0.141.0
# 2023-05-18 15:36:18 [ℹ]  using region eu-central-1
# 2023-05-18 15:36:18 [ℹ]  setting availability zones to [eu-central-1c eu-central-1b eu-central-1a]
# 2023-05-18 15:36:18 [ℹ]  subnets for eu-central-1c - public:192.168.0.0/19 private:192.168.96.0/19
# 2023-05-18 15:36:18 [ℹ]  subnets for eu-central-1b - public:192.168.32.0/19 private:192.168.128.0/19
# 2023-05-18 15:36:18 [ℹ]  subnets for eu-central-1a - public:192.168.64.0/19 private:192.168.160.0/19
# 2023-05-18 15:36:18 [ℹ]  nodegroup "demo-nodes" will use "" [AmazonLinux2/1.26]
# 2023-05-18 15:36:18 [ℹ]  using Kubernetes version 1.26
# 2023-05-18 15:36:18 [ℹ]  creating EKS cluster "demo-cluster" in "eu-central-1" region with managed nodes
# 2023-05-18 15:36:18 [ℹ]  will create 2 separate CloudFormation stacks for cluster itself and the initial managed nodegroup
# 2023-05-18 15:36:18 [ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=eu-central-1 --cluster=demo-cluster'
# 2023-05-18 15:36:18 [ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "demo-cluster" in "eu-central-1"
# 2023-05-18 15:36:18 [ℹ]  CloudWatch logging will not be enabled for cluster "demo-cluster" in "eu-central-1"
# 2023-05-18 15:36:18 [ℹ]  you can enable it with 'eksctl utils update-cluster-logging --enable-types={SPECIFY-YOUR-LOG-TYPES-HERE (e.g. all)} --region=eu-central-1 --cluster=demo-cluster'
# 2023-05-18 15:36:18 [ℹ]  
# 2 sequential tasks: { create cluster control plane "demo-cluster", 
#     2 sequential sub-tasks: { 
#         wait for control plane to become ready,
#         create managed nodegroup "demo-nodes",
#     } 
# }
# 2023-05-18 15:36:18 [ℹ]  building cluster stack "eksctl-demo-cluster-cluster"
# 2023-05-18 15:36:19 [ℹ]  deploying stack "eksctl-demo-cluster-cluster"
# 2023-05-18 15:36:49 [ℹ]  waiting for CloudFormation stack "eksctl-demo-cluster-cluster"
# 2023-05-18 15:37:19 [ℹ]  waiting for CloudFormation stack "eksctl-demo-cluster-cluster"
# 2023-05-18 15:38:19 [ℹ]  waiting for CloudFormation stack "eksctl-demo-cluster-cluster"
# 2023-05-18 15:39:20 [ℹ]  waiting for CloudFormation stack "eksctl-demo-cluster-cluster"
# 2023-05-18 15:40:20 [ℹ]  waiting for CloudFormation stack "eksctl-demo-cluster-cluster"
# 2023-05-18 15:41:20 [ℹ]  waiting for CloudFormation stack "eksctl-demo-cluster-cluster"
# 2023-05-18 15:42:20 [ℹ]  waiting for CloudFormation stack "eksctl-demo-cluster-cluster"
# 2023-05-18 15:43:20 [ℹ]  waiting for CloudFormation stack "eksctl-demo-cluster-cluster"
# 2023-05-18 15:44:21 [ℹ]  waiting for CloudFormation stack "eksctl-demo-cluster-cluster"
# 2023-05-18 15:45:21 [ℹ]  waiting for CloudFormation stack "eksctl-demo-cluster-cluster"
# 2023-05-18 15:46:21 [ℹ]  waiting for CloudFormation stack "eksctl-demo-cluster-cluster"
# 2023-05-18 15:47:21 [ℹ]  waiting for CloudFormation stack "eksctl-demo-cluster-cluster"
# 2023-05-18 15:48:21 [ℹ]  waiting for CloudFormation stack "eksctl-demo-cluster-cluster"
# 2023-05-18 15:50:23 [ℹ]  building managed nodegroup stack "eksctl-demo-cluster-nodegroup-demo-nodes"
# 2023-05-18 15:50:24 [ℹ]  deploying stack "eksctl-demo-cluster-nodegroup-demo-nodes"
# 2023-05-18 15:50:24 [ℹ]  waiting for CloudFormation stack "eksctl-demo-cluster-nodegroup-demo-nodes"
# 2023-05-18 15:50:54 [ℹ]  waiting for CloudFormation stack "eksctl-demo-cluster-nodegroup-demo-nodes"
# 2023-05-18 15:51:33 [ℹ]  waiting for CloudFormation stack "eksctl-demo-cluster-nodegroup-demo-nodes"
# 2023-05-18 15:53:07 [ℹ]  waiting for CloudFormation stack "eksctl-demo-cluster-nodegroup-demo-nodes"
# 2023-05-18 15:53:49 [ℹ]  waiting for CloudFormation stack "eksctl-demo-cluster-nodegroup-demo-nodes"
# 2023-05-18 15:54:29 [ℹ]  waiting for CloudFormation stack "eksctl-demo-cluster-nodegroup-demo-nodes"
# 2023-05-18 15:55:53 [ℹ]  waiting for CloudFormation stack "eksctl-demo-cluster-nodegroup-demo-nodes"
# 2023-05-18 15:55:53 [ℹ]  waiting for the control plane to become ready
# 2023-05-18 15:55:54 [✔]  saved kubeconfig as "/Users/fsiegrist/.kube/config"
# 2023-05-18 15:55:54 [ℹ]  no tasks
# 2023-05-18 15:55:54 [✔]  all EKS cluster resources for "demo-cluster" have been created
# 2023-05-18 15:55:54 [ℹ]  nodegroup "demo-nodes" has 2 node(s)
# 2023-05-18 15:55:54 [ℹ]  node "ip-192-168-48-96.eu-central-1.compute.internal" is ready
# 2023-05-18 15:55:54 [ℹ]  node "ip-192-168-64-248.eu-central-1.compute.internal" is ready
# 2023-05-18 15:55:54 [ℹ]  waiting for at least 1 node(s) to become ready in "demo-nodes"
# 2023-05-18 15:55:54 [ℹ]  nodegroup "demo-nodes" has 2 node(s)
# 2023-05-18 15:55:54 [ℹ]  node "ip-192-168-48-96.eu-central-1.compute.internal" is ready
# 2023-05-18 15:55:54 [ℹ]  node "ip-192-168-64-248.eu-central-1.compute.internal" is ready
# 2023-05-18 15:55:55 [ℹ]  kubectl command should work with "/Users/fsiegrist/.kube/config", try 'kubectl get nodes'
# 2023-05-18 15:55:55 [✔]  EKS cluster "demo-cluster" in "eu-central-1" region is ready

```

After nearly 20 minutes the cluster is ready. Kubectl was automatically configured to connect to the new cluster. The configuration is stored in `~/.kube/config`.

Let's review the created cluster now. 

```sh
eksctl get clusters
# NAME	        REGION        EKSCTL CREATED
# demo-cluster  eu-central-1  True

kubectl get nodes
# NAME                                              STATUS   ROLES    AGE     VERSION
# ip-192-168-48-96.eu-central-1.compute.internal    Ready    <none>   6m12s   v1.26.2-eks-a59e1f0
# ip-192-168-64-248.eu-central-1.compute.internal   Ready    <none>   6m11s   v1.26.2-eks-a59e1f0

aws iam list-roles --query "Roles[].RoleName"
# [
#     "AWSServiceRoleForAmazonEKS",
#     "AWSServiceRoleForAmazonEKSNodegroup",
#     "AWSServiceRoleForAutoScaling",
#     "AWSServiceRoleForSupport",
#     "AWSServiceRoleForTrustedAdvisor",
#     "eksctl-demo-cluster-cluster-ServiceRole-E4TVKLGKTBIZ",
#     "eksctl-demo-cluster-nodegroup-dem-NodeInstanceRole-KJW6DGERTROS"
# ]
```
