## Demo Project - Create an AWS EKS Cluster with Fargate Profile

### Topics of the Demo Project
Create EKS cluster with Fargate profile

### Technologies Used
- Kubernetes
- AWS EKS
- AWS Fargate

### Project Description
- Create Fargate IAM Role
- Create Fargate Profile
- Deploy an example application to EKS cluster using Fargate profile

#### Steps to create Fargate IAM role
- Open your browser and login to your account of the [AWS Management Console](https://eu-central-1.console.aws.amazon.com/console/home?region=eu-central-1#). 
- Open the IAM dashboard (Services > Security, Identity & Compliance > IAM) and click on Access Management > Roles in the menu on the left.
- Press the "Create role" button, select "AWS service" as the trusted entity type, select "EKS" from the dropdown at the bottom (Use cases for other AWS services), select "EKS - Fargate pod" and press the "Next" button.
- 'AmazonEKSFargatePodExecutionRolePolicy' is the only policy set. Press the "Next" button.
- Enter 'eks-fargate-role' as the role name and press "Create role".

#### Steps to create a Fargate profile
- Go to the EKS dashboard, navigate to the clusters overview, click on the cluster 'eks-cluster-test', open the "Compute" tab, scroll down to the "Fargate profiles" section and press the "Add Fargate profile" button.
- Enter 'dev-profile' as the name and select the 'eks-fargate-role' we just created.
- Below that we can select the  subnets to be used from our VPC. Even if we won't see the virtual machines provisioned by Fargate, the Pods running on these VMs will get IP addresses from our subnet IP range. Make sure only the private subnets are selected (the public subnets should not be selectable). Press "Next".
- Now we configure the pod selection rule which specifies which pods should be scheduled by Fargate and which by the Node Group. We can let Fargate schedule Pods of certain namespaces and/or having certain labels. Let's use both possibilities. Add 'dev' into the namespace textfield and add a label 'profile:fargate' (key:value). Press "Next", review your entries and press "Create".

The Fargate profile 'dev-profile' is in status "Creating" now and will change to "Active" after a few minutes.

#### Steps to deploy an example application to EKS cluster using Fargate profile
If we want our nginx Pods to be scheduled by Fargate, we have to add the namespace and label specified in the Pod selection rule to its K8s deployment configuration file. Create a new `nginx-deployment.yaml` file with the following content:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: dev # <---
spec:
  selector:
    matchLabels:
      app: nginx
      profile: fargate # <---
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
        profile: fargate # <---
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

Now execute the following commands:

```sh
# create the namespace
kubectl create namespace dev

# apply the deployment configuration
kubectl apply -f nginx-deployment.yaml

# check the pod is running
kubectl get pods -n dev -w
# NAME                     READY   STATUS              RESTARTS   AGE
# nginx-7f5bb7bcc5-x5bwh   0/1     Pending             0          7s
# nginx-7f5bb7bcc5-x5bwh   0/1     Pending             0          35s
# nginx-7f5bb7bcc5-x5bwh   0/1     ContainerCreating   0          36s
# nginx-7f5bb7bcc5-x5bwh   1/1     Running             0          43s
```

The pod was pending for 35 seconds because Fargate creates a virtual machine for each pod, which takes some time.

Now let's see the nodes:
```sh
kubectl get nodes
# NAME                                                       STATUS   ROLES    AGE     VERSION
# fargate-ip-192-168-164-158.eu-central-1.compute.internal   Ready    <none>   2m57s   v1.26.3-eks-f4dc2c0
# ip-192-168-222-24.eu-central-1.compute.internal            Ready    <none>   3d4h    v1.26.2-eks-a59e1f0
```

The first node is the newly created virtual machine. We don't see it in our AWS account. But still it got an IP address from the range of a subnet in our VPC. The second one is the EC2 instance created by the node group in demo project #1. We can see this instance in our AWS account.
