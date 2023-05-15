## Demo Project - Create an AWS EKS Cluster with a Node Group

### Topics of the Demo Project
Create AWS EKS cluster with a Node Group

### Technologies Used
- Kubernetes
- AWS EKS

### Project Description
- Configure necessary IAM Roles
- Create VPC with Cloudformation Template for Worker Nodes
- Create EKS cluster (Control Plane Nodes)
- Create Node Group for Worker Nodes and attach to EKS cluster
- Configure Auto-Scaling of worker nodes
- Deploy a sample application to EKS cluster

#### Steps to configure the necessary IAM roles
We create an IAM role in our AWS account and assign that role to the EKS cluster managed by AWS. This is necessary to allow AWS to create and manage components on our behalf.

- Open your browser and login to your account of the [AWS Management Console](https://eu-central-1.console.aws.amazon.com/console/home?region=eu-central-1#).
- Open the IAM dashboard (Services > Security, Identity & Compliance > IAM) and click on Access Management > Roles in the menu on the left.
- Press the "Create role" button, select "AWS service" as the trusted entity type, select "EKS" from the dropdown at the bottom (Use cases for other AWS services), select "EKS - Cluster" and press the "Next" button.
- The "AmazonEKSClusterPolicy" has been automatically selected for the chosen use case. Press the "Next" button.
- Enter `eks-cluster-role` as the name for the new role and press the "Create role" button.


#### Steps to create a VPC with Cloudformation Template for Worker Nodes
- Open the CloudFormation dashboard (Services > Management & Governance > CloudFormation) and press the orange "Create stack" button.
- Select "Template is ready" and "Amazon S3 URL" and paste the following URL into the URL field:\
https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/amazon-eks-vpc-private-subnets.yaml\
The URL can be copied from [this documentation page](https://docs.aws.amazon.com/eks/latest/userguide/creating-a-vpc.html). Press the "Next" button.
- Enter `eks-worker-node-vpc-stack` as the stack name and press the "Next" button.
- On the next page leave all fields unchanged and press the "Next" button once more.
- On the summary page press the "Submit" button.

Now the VPC stack is being created (status = CREATE_IN_PROGRESS). Press the refresh button until the status is CREATE_COMPLETE. On the "Outputs" tab you find the IDs of the new VPC, the subnets and the security group. We're going to need these IDs when creating the EKS cluster.

#### Steps to create EKS cluster (Control Plane Nodes)
- Open the EKS dashboard (Services > Containers > Elastic Kubernetes Services) in the AWS management console. (Note that EKS isn't free, so you will be charged for using it. Make sure to remove the service when you don't need it anymore.) Press the "Add cluster" button and select "Create".
- Enter 'eks-cluster-test' as the cluster name, select the Kubernetes version (e.g. 1.26), and the IAM role we defined before (eks-cluster-role). We don't enable the Secrets encryption via KMS (key management services). It would encrypt the K8s Secrets (which are only bas64 encoded) to prevent them from being read by un-authorized people. Press the "Next" button.
- Select the VPC of the 'eks-worker-node-vpc-stack' we created before. The related subnets are prefilled. Also select the security group belonging to the 'eks-worker-node-vpc-stack' we created before. In the "Cluster enpoint access" section choose "Public and private". Press the "Next" button.
- We don't need any control plane logs to be sent to CloudWatch, so just press the "Next" button.
- Don't select any additional EKS add-ons either, just press "Next" again.
- Leave the default versions of the automatically installed add-ons unchanged and press "Next" once more.
- On the Review page press the "Create" button. The status of the new EKS cluster is "Creating". Press the refresh button until it is "Active" (after ca. 10-15min).

#### Steps to connect to EKS cluster locally with kubectl
Even if we don't have any worker nodes running, we can connect to the EKS cluster using kubectl from our local machine. We create a kubeconfig file and check the connection with the following commands:
```sh
# make sure your aws configuration is set to the region of the EKS cluster
aws configure list
#       Name                    Value             Type    Location
#       ----                    -----             ----    --------
#    profile                <not set>             None    None
# access_key     ****************BDVT shared-credentials-file    
# secret_key     ****************eXn0 shared-credentials-file    
#     region             eu-central-1      config-file    ~/.aws/config

# make sure there is no old ~/.kube/config file
rm ~/.kube/config
# or
mv ~/.kube/config ~/.kube/config_backup

# now create a new ~/.kube/config file
aws eks update-kubeconfig --name eks-cluster-test
# Added new context arn:aws:eks:eu-central-1:369076538622:cluster/eks-cluster-test to ~/.kube/config

# check the connection
kubectl cluster-info
# Kubernetes control plane is running at https://73A57A23BA7BAAE56115E5F68C988976.gr7.eu-central-1.eks.amazonaws.com
# CoreDNS is running at https://73A57A23BA7BAAE56115E5F68C988976.gr7.eu-central-1.eks.amazonaws.com/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

#### Steps to create Node Group for Worker Nodes and attach to EKS cluster
**Step 1:** Create an IAM role for the Node Group\
- Go back to the AWS management console, open the IAM dashboard (Services > Security, Identity & Compliance > IAM) and click on Access Management > Roles in the menu on the left. Press the "Create role" button, select "AWS service" as the trusted entity type, select "EC2" and press "Next".
- On the "Add permissions" page, select the following policies:
  - AmazonEKSWorkerNodePolicy
  - AmazonEC2ContainerRegistryReadOnly: pull new image versions when they become available
  - AmazonEKS_CNI_Policy: Container Network Interface, K8s internal network needed for inter-pod-communication
and press "Next".
- Enter 'eks-node-group-role' as the role name, review your entries and press the "Create role" button.

**Step 2:** AddnNode group to EKS cluster\
- Go back to the EKS dashboard and open the cluster 'eks-cluster-test'. Select the "Compute" tab, scroll down to the "Node group" section and press the "Add node group" button.
- Enter 'eks-node-group' as the name, select the 'eks-node-group-role' we just created and press the "Next" button.
- Select the AMI type "Amazon Linux 2 (AL2_x86_64)", the Capacity type "On-Demand", the Instance type "t3.small" and the Disk size "20" GiB.
- Leave the default values in the "Node Group scaling configuration" section unchanged (min 2, max 2, desired 2). The same holds for the "Node group update configuration" section (number 1). Press "Next".
- Don't change the selected subnets. Toggle (enable) the "Configure remote access to nodes" switch and press "Enable" in the displayed warning dialog. Select one of the available EC2 key pairs created earlier to ssh into EC2 instances (e.g. 'docker-server') or create a new key pair if preferred. It is recommended to select a security group with a configured firewall rule restricting ssh access from the IP address of your local machine only. But for the moment we select "All" (Do not restrict source IPs that can remotely access nodes). We can change this configuration later.
- Check your entries on the review page and press "Create".

The status of the node group is now "Creating". It will take some time until the worker nodes are created (5min). On the EC2 dashboard you can already see the two new instances being in the status "Initializing".

When the instances are active, you should see them when executing
```sh
kubectl get nodes
# NAME                                              STATUS   ROLES    AGE     VERSION
# ip-192-168-177-9.eu-central-1.compute.internal    Ready    <none>   6m59s   v1.26.2-eks-a59e1f0
# ip-192-168-222-24.eu-central-1.compute.internal   Ready    <none>   7m2s    v1.26.2-eks-a59e1f0
```

If you want to scale the number of worker nodes up or down you can manually edit your node group and modify the min/max/desired values in the "Node Group scaling configuration" section.

A better way to do this is to configure an autoscaler as described in the next steps.

#### Steps to configure Auto-Scaling of worker nodes

To configure the Autoscaler we need to
- have an auto scaling group (was automatically created when we set up the EKS cluster)
- create a custom policy and attach it to the Node Group IAM Role (to allow the EC2 instances to make certain AWS API calls needed for the autoscaling feature)
- deploy the K8s Autoscaler

**Step 1:** Create a custom policy\
Go to IAM dashboard > Access management > Policies and press the "Create policy" button. Switch to the JSON view by pressing the "JSON" button. Paste the following content into the policy editor:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeLaunchConfigurations",
                "autoscaling:DescribeTags",
                "autoscaling:SetDesiredCapacity",
                "autoscaling:TerminateInstanceInAutoScalingGroup",
                "ec2:DescribeLaunchTemplateVersions"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}
```

Press "Next". On the review page enter 'node-group-autoscale-policy' as the policy name and press "Create policy".

**Step 2:** Attach the custom policy to the Node Group IAM Role\
To attach this policy to the existing node group IAM role go to IAM dashboard > Access management > Roles > eks-node-group-role > Permissions, press the "Add permissions" button and choose "Attach policies". In the "Other permissions policies" section check the custom 'node-group-autoscale-policy' created before and press the "Add permissions" button.

**Step 3:** Deploy the Autoscaler\
Execute the following commands on your local machine:
```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
# serviceaccount/cluster-autoscaler created
# clusterrole.rbac.authorization.k8s.io/cluster-autoscaler created
# role.rbac.authorization.k8s.io/cluster-autoscaler created
# clusterrolebinding.rbac.authorization.k8s.io/cluster-autoscaler created
# rolebinding.rbac.authorization.k8s.io/cluster-autoscaler created
# deployment.apps/cluster-autoscaler created

kubectl get deployment cluster-autoscaler -n kube-system
# NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
# cluster-autoscaler   1/1     1            1           70s

kubectl edit deployment cluster-autoscaler -n kube-system
# -> in metadata:annotations add the following line after 'deployment.kubernetes.io/revision: "1"':
#    'cluster-autoscaler.kubernetes.io/safe-to-evict: "false"'
# -> in spec:template:spec:containers replace '<YOUR CLUSTER NAME>' with 'eks-cluster-test'
#    and add the options '- --balance-similar-node-groups' 
#                    and '- --skip-nodes-with-system-pods=false'
# -> make sure the spec:template:spec:containers:image version matches the Kubernetes version used in the EKS cluster (1.26); get the exact tag (1.26.2) from https://github.com/kubernetes/autoscaler/tags
```

Of course you can also first download the [autoscaler configurationfile](https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml), make all the changes and then deploy it.

Let's have a look at the logs of the autoscaler pod:
```sh
kubectl get pods -n kube-system
# NAME                                  READY   STATUS    RESTARTS   AGE
# aws-node-4k2f7                        1/1     Running   0          5h27m
# aws-node-k9thp                        1/1     Running   0          5h27m
# cluster-autoscaler-7798975c7f-dmz95   1/1     Running   0          2m
# coredns-788b9c9454-5rp7t              1/1     Running   0          7h25m
# coredns-788b9c9454-m4twb              1/1     Running   0          7h25m
# kube-proxy-fdg4k                      1/1     Running   0          5h27m
# kube-proxy-rwvzc                      1/1     Running   0          5h27m

kubectl logs cluster-autoscaler-7798975c7f-dmz95 -n kube-system | less
```

You'll find entries like
```log
I0514 21:48:09.465903       1 static_autoscaler.go:541] Calculating unneeded nodes
I0514 21:48:09.465922       1 pre_filtering_processor.go:67] Skipping ip-192-168-222-24.eu-central-1.compute.internal - node group min size reached (current: 2, min: 2)
I0514 21:48:09.465938       1 pre_filtering_processor.go:67] Skipping ip-192-168-177-9.eu-central-1.compute.internal - node group min size reached (current: 2, min: 2)
I0514 21:48:09.465974       1 static_autoscaler.go:589] Scale down status: lastScaleUpTime=2023-05-14 20:43:27.79390843 +0000 UTC m=-3578.293245296 lastScaleDownDeleteTime=2023-05-14 20:43:27.79390843 +0000 UTC m=-3578.293245296 lastScaleDownFailTime=2023-05-14 20:43:27.79390843 +0000 UTC m=-3578.293245296 scaleDownForbidden=false scaleDownInCooldown=false
I0514 21:48:09.466007       1 static_autoscaler.go:598] Starting scale down
I0514 21:48:09.466066       1 legacy.go:298] No candidates for scale down
```

#### Steps to deploy a sample application to EKS cluster
**Step 1:** Create an nginx deployment config\
Create a file called `nginx.yaml` with the following content:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  type: LoadBalancer
```

**Step 2:** Apply it to the cluster
```sh
kubectl apply -f nginx.yaml
# =>
# deployment.apps/nginx created
# service/nginx created

kubectl get pods
# NAME                     READY   STATUS    RESTARTS   AGE
# nginx-7f456874f4-54dmv   1/1     Running   0          117s

kubectl get services
# NAME         TYPE           CLUSTER-IP       EXTERNAL-IP                                                                  PORT(S)        AGE
# kubernetes   ClusterIP      10.100.0.1       <none>                                                                       443/TCP        29h
# nginx        LoadBalancer   10.100.224.113   a3c0ab05fe05d4e3bb204fd409810766-1007316954.eu-central-1.elb.amazonaws.com   80:31338/TCP   2m40s
```

Creating a K8s service of type LoadBalancer automatically creates a cloud native LoadBalancer of the cluster environment too (in this case AWS EKS). As you see the cloud native LoadBalancer with the IP address 'a3c0ab05fe05d4e3bb204fd409810766-1007316954.eu-central-1.elb.amazonaws.com' (and default port 80, not displayed in the above output) forwards incoming requests to the node port 31338 which is connected to the K8s LoadBalancer service with the cluster IP address 10.100.224.113 listening on port 80. Entering the external IP address in the browser lets you access the [nginx application](http://a3c0ab05fe05d4e3bb204fd409810766-1007316954.eu-central-1.elb.amazonaws.com).
