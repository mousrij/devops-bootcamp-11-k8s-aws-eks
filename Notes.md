## Notes on the videos for Module 11 "Kubernetes on AWS - EKS"
<br />

<details>
<summary>Video: 1 - Container Services on AWS</summary>
<br />

There are multiple options for running a containerized application on AWS:
- Elastic Container Service (ECS): Container orchestration service
- Elastic Kubernetes Service (EKS): Managed Kubernetes Service
- Elastic Container Registry (ECR): Private Docker Repository

### Elastic Container Service (ECS)
Amazon's Elastic Container Service is one of several container orchestration tools (like Docker Swarm, Kubernetes, Apache Mesos, Hashicorp Nomad). It manages the whole container lifecycle (start, re-schedule, load balance).

An ESC cluster contains all the services to manage the containers. It represents a control plane for all the virtual machines (EC2 servers) that are running containers. The EC2 instances are not isolated but connected to the ECS cluster and managed by its control plane. On each EC2 instance there is a container runtime and an ECS agent (for communication with the control plane).

It is still your job to create the EC2 instances, join them to the ECS cluster, check whether they provide enough recources for the containers, manage the operating system (updates, patches), care for the container runtime and the ECS agent.

If you want to delegate the management of the infrastructure to AWS too, you can use AWS Fargate, which is a serverless way to launch containers. You don't have to provision and manage the server yourself. Each time you want to run a new container you hand it over to Fargate which will analyze its resource requirements and provision a server matching these requirements on demand. You pay only for what you use, not a whole EC2 instance which probably isn't fully used.

### Elastic Kubernetes Service (EKS)
If you want to use Kubernetes as your container orchestration tool, AWS provides EKS.

Difference between ECS and Kubernetes:
- ECS is specific to AWS, difficult to migrate
- ECS is less complex and its control plane is free
- K8s is open source, easier to migrate to another platform (if you don't use to many other AWS services)

When you create an EKS cluster, AWS will provision Kubernetes master nodes, with all the needed K8s control plane services already installed. They will be replicated in multiple availability zones of the chosen region. AWS will also manage (replicate in multiple availability zones, backup) the etcd storage components.

For the worker nodes you need to create and manage EC2 instances (the so called compute fleed) and connect them to the EKS cluster. A semi-managed variant is using EKS with node group(s), where the EC2 instances are managed for you. All the processes needed on K8s worker nodes (like container runtime, K8s agent, etc.) will be installed on them. But you still have to configure the nodegroups (e.g. scaling behavior). As with ECS it is also possible to combine EKS with Fargate, resulting in fully managed worker nodes.

To create an EKS cluster, you have to 
- provision an EKS cluster (Control Plane Nodes)
- create a node group of EC2 instances (Worker Nodes)
- connect the node group(s) to the EKS cluster
- deploy your containerized applications

### Elastic Container Registry (ECR)
ECR is the AWS repository for Docker images (as an alternative to Docker Hub or Nexus). Of course it integrates very well with other AWS services.

</details>

*****

<details>
<summary>Video: 2 - Create EKS cluster with AWS Management Console</summary>
<br />

Steps to create an EKS cluster:
- create an IAM role for the EKS cluster
- create a VPC for the EKS worker nodes
- create an EKS cluster
- connect kubectl with the EKS cluster
- create an EC2 IAM role for the node group
- create a node group and attach it to the EKS cluster
- configure auto-scaling
- deploy your application to the EKS cluster

Also check the [documentation](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-console.html).

### Create an IAM Role for the EKS Cluster
We create an IAM role in our AWS account and assign that role to the EKS cluster managed by AWS. This is necessary to allow AWS to create and manage components on our behalf.

Open your browser and login to your account of the [AWS Management Console](https://eu-central-1.console.aws.amazon.com/console/home?region=eu-central-1#). Open the IAM dashboard (Services > Security, Identity & Compliance > IAM) and click on Access Management > Roles in the menu on the left. Press the "Create role" button, select "AWS service" as the trusted entity type, select "EKS" from the dropdown at the bottom (Use cases for other AWS services), select "EKS - Cluster" and press the "Next" button. The "AmazonEKSClusterPolicy" has been automatically selected for the chosen use case. Press the "Next" button. Enter a unique name for the role (e.g. eks-cluster-role) and press the "Create role" button.

### Create a VPC for the EKS Worker Nodes
Each AWS account has a default VPC. So why do we need another VPC for our EKS cluster? An EKS cluster needs specific networking configuration. The worker nodes need specific firewall configurations for the communication with the control plane. Best practices suggest configuring a public subnet (for a cloudnative loadbalancer, e.g. Elastic Load Balancer) and a private subnet (for the K8s LoadBalancer service) (check the [documentation](https://docs.aws.amazon.com/eks/latest/userguide/creating-a-vpc.html)). Through an IAM role we give K8s permission to change VPC configurations. These should not affect the default VPC.

However, we don't have to configure the new VPC and all the required components by ourselves. Instead we can use the cloudformation template, with which the whole stack of VPC and required components suitable for EKS is created. See [VPC Cloudformation Template](https://docs.aws.amazon.com/codebuild/latest/userguide/cloudformation-vpc-template.html).

Open the CloudFormation dashboard (Services > Management & Governance > CloudFormation) and press the orange "Create stack" button. Select "Template is ready" and "Amazon S3 URL" and paste the following URL into the URL field:

https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/amazon-eks-vpc-private-subnets.yaml

The URL can be copied from [this documentation page](https://docs.aws.amazon.com/eks/latest/userguide/creating-a-vpc.html). You may also enter that URL in the browser to download the template file and have a look at it. You could adjust it and upload it. For our purposes the default template file is fine, so we press the "Next" button.

Enter a stack name (e.g. eks-worker-node-vpc-stack) and press the "Next" button. On the next page leave all fields unchanged and press the "Next" button once more. On the summary page press the "Submit" button.

Now the VPC stack is being created (status = CREATE_IN_PROGRESS). Press the refresh button until the status is CREATE_COMPLETE. On the "Outputs" tab you find the IDs of the new VPC, the subnets and the security group. We're going to need these IDs when creating the EKS cluster.

### Create the EKS Cluster
Open the EKS dashboard (Services > Containers > Elastic Kubernetes Services) in the AWS management console. (Note that EKS isn't free, so you will be charged for using it. Make sure to remove the service when you don't need it anymore.) Press the "Add cluster" button and select "Create".

Enter a cluster name (e.g. eks-cluster-test), select the Kubernetes version (e.g. 1.26), and the IAM role we defined before (eks-cluster-role). We don't enable the Secrets encryption via KMS (key management services). It would encrypt the K8s Secrets (which are only bas64 encoded) to prevent them from being read by un-authorized people. Press the "Next" button.

Select the VPC of the eks-worker-node-vpc-stack we created before. The related subnets are prefilled. Also select the security group belonging to the eks-worker-node-vpc-stack we created before. In the "Cluster enpoint access" section choose "Public and private". We want to access the cluster (e.g. via kubectl) from our local machine (public), but the control plane should communicate with the worker nodes only within the VPC (private). Press the "Next" button.

We don't need any control plane logs to be sent to CloudWatch, so just press the "Next" button. Don't select any additional EKS add-ons either, just press "Next" again. Leave the default versions of the automatically installed add-ons unchanged and press "Next" once more.

On the Review page press the "Create" button. The status of the new EKS cluster is "Creating". Press the refresh button until it is "Active" (after ca. 10-15min).

### Connect to EKS Cluster locally with kubectl
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

### Create an EC2 IAM Role for our Node Group
Kubelet is the main worker process running on worker nodes. It is responsible for scheduling and managing Kubernetes components like Pods and must be able to communicate with the Control Plane or other AWS services. That's why Kubelet needs according permissions to do its job.

So let's create an IAM role for the Node Group. With Node Group all necessary worker processes likecontainer runtime, kubelet, k-proxy etc. are installed.

Go back to the AWS management console, open the IAM dashboard (Services > Security, Identity & Compliance > IAM) and click on Access Management > Roles in the menu on the left. Press the "Create role" button, select "AWS service" as the trusted entity type, select "EC2" and press "Next".

On the "Add permissions" page, select the following policies:
- AmazonEKSWorkerNodePolicy
- AmazonEC2ContainerRegistryReadOnly: pull new image versions when they become available
- AmazonEKS_CNI_Policy: Container Network Interface, K8s internal network needed for inter-pod-communication
and press "Next".

Enter a role name (e.g. eks-node-group-role), review your entries and press the "Create role" button.

### Add Node Group to EKS Cluster
Go back to the EKS dashboard and open the cluster 'eks-cluster-test'. Select the "Compute" tab, scroll down to the "Node group" section and press the "Add node group" button. Enter a name (e.g. eks-node-group), select the 'eks-node-group-role' we just created and press the "Next" button.

Select the AMI type "Amazon Linux 2 (AL2_x86_64)", the Capacity type "On-Demand", the Instance type "t3.small" and the Disk size "20" GiB.

Leave the default values in the "Node Group scaling configuration" section unchanged (min 2, max 2, desired 2). The same holds for the "Node group update configuration" section (number 1). Press "Next".

Don't change the selected subnets. Toggle (enable) the "Configure remote access to nodes" switch and press "Enable" in the displayed warning dialog. Select one of the available EC2 key pairs created earlier to ssh into EC2 instances (e.g. docker-server) or create a new key pair if preferred. It is recommended to select a security group with a configured firewall rule restricting ssh access from the IP address of your local machine only. But for the moment we select "All" (Do not restrict source IPs that can remotely access nodes). We can change this configuration later.

Check your entries on the review page and press "Create". The status of the node group is now "Creating". It will take some time until the worker nodes are created (5min). On the EC2 dashboard you can already see the two new instances being in the status "Initializing".

When the instances are active, you should see them when executing
```sh
kubectl get nodes
# NAME                                              STATUS   ROLES    AGE     VERSION
# ip-192-168-177-9.eu-central-1.compute.internal    Ready    <none>   6m59s   v1.26.2-eks-a59e1f0
# ip-192-168-222-24.eu-central-1.compute.internal   Ready    <none>   7m2s    v1.26.2-eks-a59e1f0
```

If you want to scale the number of worker nodes up or down you can manually edit your node group and modify the min/max/desired values in the "Node Group scaling configuration" section.

A better way to do this is to configure an autoscaler as will be demonstrated in the next video.

</details>

*****

<details>
<summary>Video: 3 - Configure Autoscaling in EKS cluster</summary>
<br />

With creating an EKS cluster, an auto scaling group was automatically created (see "EC2 dashboard > Auto Scaling groups" or "EKS dashboard > Clusters > eks-cluster-test > Compute > Node groups > eks-node-group > Details > Autoscaling group name"). However this component just groups the EC2 instances together. It does not autoscale the resources within this group. We need to configure the K8s Autoscaler component to work together with the auto scaling group. The K8s Autoscaler will then add or remove EC2 instances depending on the workload, but only within the range (min, max, desired) defined for the auto scaling group.

To configure the Autoscaler we need to
- have an auto scaling group (was automatically created when we set up the EKS cluster)
- create a custom policy and attach it to the Node Group IAM Role (to allow the EC2 instances to make certain AWS API calls needed for the autoscaling feature)
- deploy the K8s Autoscaler

### Create a custom policy
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

Press "Next". On the review page enter a policy name (e.g. node-group-autoscale-policy) and press "Create policy".

To attach this policy to the existing node group IAM role go to IAM dashboard > Access management > Roles > eks-node-group-role > Permissions, press the "Add permissions" button and choose "Attach policies". In the "Other permissions policies" section check the custom node-group-autoscale-policy created before and press the "Add permissions" button.

### Deploy the K8s Autoscaler
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

Let's adjust the min/max values to see the autoscaler in action. Go to the EC2 dashboard, click on the "Auto Scaling Groups" link, click on the eks-node-group autoscaling group and press the "Edit" button in the "Group details" section. Set the Minimum capacity to 1 and the Maximum capacity to 3 and press the "Update" button.

The autoscaler gets informed about the new values and checks the status of the nodes during the next 10 minutes. Then it starts removing one node.

```sh
kubectl get nodes
# NAME                                              STATUS   ROLES    AGE   VERSION
# ip-192-168-177-9.eu-central-1.compute.internal    Ready    <none>   27h   v1.26.2-eks-a59e1f0
# ip-192-168-222-24.eu-central-1.compute.internal   Ready    <none>   27h   v1.26.2-eks-a59e1f0

kubectl logs -f cluster-autoscaler-7798975c7f-dmz95 -n kube-system  
# I0515 19:40:20.180533       1 nodes.go:123] ip-192-168-177-9.eu-central-1.compute.internal was unneeded for 9m51.772869583s
# I0515 19:40:20.180542       1 legacy.go:298] No candidates for scale down
# ...
# I0515 19:40:30.270914       1 nodes.go:123] ip-192-168-177-9.eu-central-1.compute.internal was unneeded for 10m1.783173228s
# I0515 19:40:30.283285       1 delete.go:103] Successfully added ToBeDeletedTaint on node ip-192-168-177-9.eu-central-1.compute.internal
# I0515 19:40:30.283570       1 actuator.go:161] Scale-down: removing empty node "ip-192-168-177-9.eu-central-1.compute.internal"
# I0515 19:40:30.284386       1 actuator.go:244] Scale-down: waiting 5s before trying to delete nodes
# ...
# I0515 19:40:35.451612       1 auto_scaling_groups.go:311] Terminating EC2 instance: i-02580710e75f1f082

kubectl get nodes
# NAME                                              STATUS                     ROLES    AGE   VERSION
# ip-192-168-177-9.eu-central-1.compute.internal    Ready,SchedulingDisabled   <none>   27h   v1.26.2-eks-a59e1f0
# ip-192-168-222-24.eu-central-1.compute.internal   Ready                      <none>   27h   v1.26.2-eks-a59e1f0

kubectl get nodes
# NAME                                              STATUS   ROLES    AGE   VERSION
# ip-192-168-222-24.eu-central-1.compute.internal   Ready    <none>   27h   v1.26.2-eks-a59e1f0
```

### Deploy an nginx Application with LoadBalancer
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

Apply it to the cluster:
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

Creating a K8s service of type LoadBalancer automatically creates a cloud native LoadBalancer of the cluster environment too (in this case AWS EKS). As you see the cloud native LoadBalancer with the IP address 'a3c0ab05fe05d4e3bb204fd409810766-1007316954.eu-central-1.elb.amazonaws.com' (and default port 80, not displayed in the above output) forwards incoming requests to the node port 31338 which is connected to the K8s LoadBalancer service with the cluster IP address 10.100.224.113 listening on port 80. Entering the external IP address in the browser lets you access the nginx application.

### 20 Replicas - Autoscaler in Action
Let's increase the number of nginx replicas to 20 to see the autoscaler launch new worker nodes.

```sh
kubectl scale deployment nginx --replicas=20
# deployment.apps/nginx scaled

kubectl logs -f cluster-autoscaler-7798975c7f-dmz95 -n kube-system 
# I0515 20:27:08.660248       1 hinting_simulator.go:110] failed to find place for default/nginx-7f456874f4-9jdrf based on similar pods scheduling
# I0515 20:27:08.660293       1 hinting_simulator.go:110] failed to find place for default/nginx-7f456874f4-mp8m4 based on similar pods scheduling
# I0515 20:27:08.660335       1 hinting_simulator.go:110] failed to find place for default/nginx-7f456874f4-5w4hb based on similar pods scheduling
# I0515 20:27:08.660377       1 hinting_simulator.go:110] failed to find place for default/nginx-7f456874f4-2szgh based on similar pods scheduling
# I0515 20:27:08.660419       1 hinting_simulator.go:110] failed to find place for default/nginx-7f456874f4-fzthv based on similar pods scheduling
# I0515 20:27:08.660459       1 hinting_simulator.go:110] failed to find place for default/nginx-7f456874f4-z8bwh based on similar pods scheduling
# I0515 20:27:08.660502       1 hinting_simulator.go:110] failed to find place for default/nginx-7f456874f4-k6sjg based on similar pods scheduling
# I0515 20:27:08.660545       1 hinting_simulator.go:110] failed to find place for default/nginx-7f456874f4-c2hrb based on similar pods scheduling
# I0515 20:27:08.660593       1 hinting_simulator.go:110] failed to find place for default/nginx-7f456874f4-bs5hl based on similar pods scheduling
# I0515 20:27:08.660631       1 hinting_simulator.go:110] failed to find place for default/nginx-7f456874f4-rb2ld based on similar pods scheduling
# I0515 20:27:08.660671       1 hinting_simulator.go:110] failed to find place for default/nginx-7f456874f4-hnb2d based on similar pods scheduling
# I0515 20:27:08.660710       1 hinting_simulator.go:110] failed to find place for default/nginx-7f456874f4-j9mnh based on similar pods scheduling
# I0515 20:27:08.660747       1 hinting_simulator.go:110] failed to find place for default/nginx-7f456874f4-rn7xc based on similar pods scheduling
# ...
# I0515 20:27:08.661732       1 scale_up.go:282] Best option to resize: eks-eks-node-group-e0c40d85-c6a1-2ad5-0296-40386965ef34
# I0515 20:27:08.661743       1 scale_up.go:286] Estimated 2 nodes needed in eks-eks-node-group-e0c40d85-c6a1-2ad5-0296-40386965ef34
# I0515 20:27:08.661769       1 scale_up.go:405] Final scale-up plan: [{eks-eks-node-group-e0c40d85-c6a1-2ad5-0296-40386965ef34 1->3 (max: 3)}]
# I0515 20:27:08.661791       1 scale_up.go:608] Scale-up: setting group eks-eks-node-group-e0c40d85-c6a1-2ad5-0296-40386965ef34 size to 3
# I0515 20:27:08.661845       1 auto_scaling_groups.go:248] Setting asg eks-eks-node-group-e0c40d85-c6a1-2ad5-0296-40386965ef34 size to 3
# ...
# I0515 20:27:28.824124       1 filter_out_schedulable.go:120] 14 pods marked as unschedulable can be scheduled.
# ...

kubcetl get nodes
# NAME                                              STATUS   ROLES    AGE     VERSION
# ip-192-168-222-24.eu-central-1.compute.internal   Ready    <none>   28h     v1.26.2-eks-a59e1f0
# ip-192-168-35-232.eu-central-1.compute.internal   Ready    <none>   6m46s   v1.26.2-eks-a59e1f0
# ip-192-168-39-15.eu-central-1.compute.internal    Ready    <none>   6m50s   v1.26.2-eks-a59e1f0
```

</details>

*****

<details>
<summary>Video: 4 - Create Fargate Profile for EKS Cluster</summary>
<br />

With Fargate you let AWS manage the worker nodes too. You won't create any EC2 instances in your account. An important difference between Fargate and creating your own EC2 worker nodes is, that Fargate will create one virtual machine per Pod resulting in some limitation with using Fargate:
- there is no support for stateful applications yet
- there is no support for Daemon Sets (applications running on every node)

Note that we can have both Fargate and Node Group atached to our EKS cluster.

### Create an IAM Role for Fargate
Kubelet on servers provisioned by Fargate need to call AWS services, pull the container images from ECR etc. So just as we did for the EC2 instances in the Node Group we need to create a role for the Fargate servers and attach the required permissions to it.

Open your browser and login to your account of the [AWS Management Console](https://eu-central-1.console.aws.amazon.com/console/home?region=eu-central-1#). Open the IAM dashboard (Services > Security, Identity & Compliance > IAM) and click on Access Management > Roles in the menu on the left. Press the "Create role" button, select "AWS service" as the trusted entity type, select "EKS" from the dropdown at the bottom (Use cases for other AWS services), select "EKS - Fargate pod" and press the "Next" button.

'AmazonEKSFargatePodExecutionRolePolicy' is the only policy set. Press the "Next" button.

Enter a role name (e.g. 'eks-fargate-role') and press "Create role".

### Create Fargate Profile
A Fargate profile creates a Pod selection rule which defines how new pods should be scheduled. If for example there is also a node group, the selection rule specifies which pod should be scheduled by Fargate and which by the node group.

To create a Fargate profile go to the EKS dashboard, navigate to the clusters overview, click on the cluster 'eks-cluster-test', open the "Compute" tab, scroll down to the "Fargate profiles" section and press the "Add Fargate profile" button.

Enter a name (e.g. dev-profile) and select the 'eks-fargate-role' we just created. Below that we can select the  subnets to be used from our VPC. Even if we won't see the virtual machines provisioned by Fargate, the Pods running on these VMs will get IP addresses from our subnet IP range. Make sure only the private subnets are selected (the public subnets should not be selectable). Press "Next".

Now we configure the pod selection rule mentioned before. We can let Fargate schedule Pods of certain namespaces and/or having certain labels. Let's use both possibilities. Add 'dev' into the namespace textfield and add a label 'profile:fargate' (key:value). Press "Next", review your entries and press "Create". The Fargate profile 'dev-profile' is in status "Creating" now and will change to "Active" after a few minutes.

### Deploy first Pod through Fargate
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

For illustrating purposes let's change the namespace in the nginx-deployment.yaml to `default`, set the replicas to `2` and the deployment name to `nginx-test` (because we already have an nginx deployment in the default namespace).

Re-apply the configuration:
```sh
kubectl apply -f nginx-deployment.yaml

# check the pods are running
kubectl get pods -n default -o wide
# NAME                          READY   STATUS    RESTARTS   AGE   IP                NODE                                              NOMINATED NODE   READINESS GATES
# nginx-7f456874f4-2pxn7        1/1     Running   0          2d    192.168.250.72    ip-192-168-222-24.eu-central-1.compute.internal   <none>           <none>
# nginx-test-7f5bb7bcc5-6z5cw   1/1     Running   0          7s    192.168.222.107   ip-192-168-222-24.eu-central-1.compute.internal   <none>           <none>
# nginx-test-7f5bb7bcc5-gprnf   1/1     Running   0          7s    192.168.210.122   ip-192-168-222-24.eu-central-1.compute.internal   <none>           <none>
```

As we can see all three pods (the old one created in demo project #1 and the two new replicas created just now) are running on the same node (with IP address 192-168-222-24).

And now let's change the namespace back to `dev` and the deployment name to `nginx-dev` and re-apply it:
```sh
kubectl apply -f nginx-deployment.yaml

# check the pods are running
kubectl get pods -n dev -w
# NAME                         READY   STATUS              RESTARTS   AGE
# nginx-7f5bb7bcc5-x5bwh       1/1     Running             0          25m
# nginx-dev-7f5bb7bcc5-m4g8r   0/1     Pending             0          13s
# nginx-dev-7f5bb7bcc5-wmhgd   0/1     Pending             0          13s
# nginx-dev-7f5bb7bcc5-m4g8r   0/1     Pending             0          37s
# nginx-dev-7f5bb7bcc5-wmhgd   0/1     Pending             0          37s
# nginx-dev-7f5bb7bcc5-m4g8r   0/1     ContainerCreating   0          37s
# nginx-dev-7f5bb7bcc5-wmhgd   0/1     ContainerCreating   0          37s
# nginx-dev-7f5bb7bcc5-wmhgd   1/1     Running             0          44s
# nginx-dev-7f5bb7bcc5-m4g8r   1/1     Running             0          46s

kubectl get pods -n dev -o wide
# NAME                         READY   STATUS    RESTARTS   AGE    IP                NODE                                                       NOMINATED NODE   READINESS GATES
# nginx-7f5bb7bcc5-x5bwh       1/1     Running   0          27m    192.168.164.158   fargate-ip-192-168-164-158.eu-central-1.compute.internal   <none>           <none>
# nginx-dev-7f5bb7bcc5-m4g8r   1/1     Running   0          107s   192.168.183.201   fargate-ip-192-168-183-201.eu-central-1.compute.internal   <none>           <none>
# nginx-dev-7f5bb7bcc5-wmhgd   1/1     Running   0          107s   192.168.159.165   fargate-ip-192-168-159-165.eu-central-1.compute.internal   <none>           <none>

kubectl get nodes
# NAME                                                       STATUS   ROLES    AGE     VERSION
# fargate-ip-192-168-159-165.eu-central-1.compute.internal   Ready    <none>   3m48s   v1.26.3-eks-f4dc2c0
# fargate-ip-192-168-164-158.eu-central-1.compute.internal   Ready    <none>   29m     v1.26.3-eks-f4dc2c0
# fargate-ip-192-168-183-201.eu-central-1.compute.internal   Ready    <none>   3m48s   v1.26.3-eks-f4dc2c0
# ip-192-168-222-24.eu-central-1.compute.internal            Ready    <none>   3d4h    v1.26.2-eks-a59e1f0
```

We see that all the three pods in the 'dev' namespace are running on three different nodes.

### Cleanup Cluster Resources
When we want to delete the EKS cluster we first have to delete the Node Group(s) and Fargate Profile(s) attached to it. Go to the EKS dashboard, navigate to the clusters overview, click on the cluster 'eks-cluster-test', open the "Compute" tab, scroll down to the "Node groups" and "Fargate profiles" sections, select the group/profile you want to delete and press the "Delete" button". As soon as all Node Groups and Fargate Profiles attached to the cluster are deleted (which may take some time) you can delete the cluster itself. Press the "Delete cluster" button.

Once the cluster has been deleted, we can delete the three roles 'eks-cluster-role', 'eks-node-group-role' and 'eks-fargate-role'. Go to the IAM dashboard, open the roles overview and delete the three roles. The custom 'node-group-autoscale-policy' won't be deleted by this. If you wanted to delete it too, you would have to do it separately.

</details>

*****

<details>
<summary>Video: 5 - Create EKS cluster with eksctl command line tool</summary>
<br />

Manually creating an EKS cluster using the AWS Management Console is a rather inefficient way of doing it. Using AWS CLI we could do the same and assemble the commands in a script to reduce the work for doing it repeatedly. But the most efficient way of creating an EKS cluster is using the `eksctl` command line tool which automates many individual tasks. It allows to create a cluster with just one command. All the necessary components are created and configured in the background. CLI options let you customize the cluster to be created.

### Install eksctl on a Mac M2
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

Or using homebrew:
```sh
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
```

### Connect eksctl With AWS Account
If you already have configured credentials for awscli you can use the same configuration for eksctl. If you haven't you must do it first. We have to tell eksctl with which account and which user we want to connect. Get the file, that was downloaded when you created the access key for the admin user and execute the following command:

```sh
aws configure
  AWS Access Key ID [None]: # enter the AWS access key id from the downloaded .csv file
  AWS Secret Access Key [None]: # enter the AWS secret access key from the downloaded .csv file
  Default region name [None]: eu-central-1 # Frankfurt (eu-west-3 for Paris)
  Default output format [None]: json
```

This configuration will be used for all subsequent eksctl (or awscli) commands. The configuration itself is stored in `~/.aws/config` and `~/.aws/credentials`.

### Create an EKS Cluster
You can either use the `eksctl create cluster` command with all the configuration options you need, or you can write a configuration yaml file and apply it using the command `eksctl create cluster -f <config.yaml>`. Examples of configuration files can be found [here](https://github.com/weaveworks/eksctl/tree/main/examples).

For this demo we use the command options, so execute the following command:
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
# ...
# 2023-05-18 15:55:55 [ℹ]  kubectl command should work with "/Users/fsiegrist/.kube/config", try 'kubectl get nodes'
# 2023-05-18 15:55:55 [✔]  EKS cluster "demo-cluster" in "eu-central-1" region is ready

```

As you see this command takes nearly 20 minutes to complete. Kubectl was automatically configured to connect to the created cluster. The configuration was stored in `~/.kube/config`. If you had other cluster configured in this file, the configuration for the new cluster was just added. So you don't lose existing configurations.

Let's review the created cluster now. 

```sh
eksctl get clusters
# NAME	        REGION        EKSCTL CREATED
# demo-cluster  eu-central-1  True

kubectl get nodes
# NAME                                              STATUS   ROLES    AGE     VERSION
# ip-192-168-48-96.eu-central-1.compute.internal    Ready    <none>   6m12s   v1.26.2-eks-a59e1f0
# ip-192-168-64-248.eu-central-1.compute.internal   Ready    <none>   6m11s   v1.26.2-eks-a59e1f0
```

Login to your account in the AWS Management Console and check the following resources:
- IAM roles
- VPCs and Subnets
- EKS

### Links
- [eksctl.io](https://eksctl.io)
- [eksctl installation guide](https://github.com/weaveworks/eksctl#installation)
- [eksctl getting started](https://eksctl.io/introduction/#getting-started)

</details>

*****

<details>
<summary>Video: 6 - Deploy to EKS Cluster from Jenkins Pipeline</summary>
<br />

The following steps are needed to be able to deploy to our EKS demo cluster created in the last video from a Jenkins pipeline:
- install kubectl command line tool inside Jenkins container
- install aws-iam-authenticator tool inside Jenkins container (Jenkins does not only have to authenticate against the EKS cluster but also against AWS; the aws-iam-authenticator tool was installed on our local machine when we created the EKS cluster using the eksctl command)
- create a kubeconfig file to connect to the EKS cluster
- add AWS credentials (AWS user and secret access key) on Jenkins for AWS account authentication
- adjust Jenkinsfile to configure EKS cluster deployment

### Install kubectl on Jenkins Server
SSH into the DigitalOcean droplet where Jenkins is running:
```sh
ssh root@<jenkins-droplet-ip>

# make sure jenkins is running
docker ps

# enter the container as root to be able to install kubectl
docker exec -u 0 -it <jenkins-container-id> bash

# get the current command to install kubectl from https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

# download the latest kubectl release
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# download the kubectl checksum file
curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"

# validate the kubectl binary against the checksum file
echo "$(cat kubectl.sha256) kubectl" | sha256sum --check
# => kubectl: OK

# install kubectl
install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# test to ensure the version you installed is up-to-date
kubectl version --output=yaml

# remove no longer needed files
rm kubectl
rm kubectl.sha256
```

### Install AWS IAM Authenticator
SSH into the DigitalOcean droplet where Jenkins is running:
```sh
ssh root@<jenkins-droplet-ip>

# make sure jenkins is running
docker ps

# enter the container as root to be able to install aws-iam-authenticator
docker exec -u 0 -it <jenkins-container-id> bash

# get the current command to install aws-iam-authenticator from https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html

# Download the aws-iam-authenticator binary
curl -Lo aws-iam-authenticator https://github.com/kubernetes-sigs/aws-iam-authenticator/releases/download/v0.5.9/aws-iam-authenticator_0.5.9_linux_amd64

# download the sha256 checksum file
curl -Lo aws-iam-authenticator.txt https://github.com/kubernetes-sigs/aws-iam-authenticator/releases/download/v0.5.9/authenticator_0.5.9_checksums.txt

# validate the kubectl binary against the checksum file
echo "$(awk '/aws-iam-authenticator_0.5.9_linux_amd64/ {print $1}' aws-iam-authenticator.txt) aws-iam-authenticator" | sha256sum --check
# => aws-iam-authenticator: OK

# install aws-iam-authenticator
install -o root -g root -m 0755 aws-iam-authenticator /usr/local/bin/aws-iam-authenticator

# test that the aws-iam-authenticator binary works
aws-iam-authenticator help

# remove no longer needed files
rm aws-iam-authenticator
rm aws-iam-authenticator.txt
```

### Create a kubeconfig File for Jenkins
Inside the Jenkins Docker container we don't have an editor available, so we create the kubeconfig file outside the container (on the DigitalOcean droplet) and copy it into the container.

But to get the content of the file, we create it on our local machine where aws-cli is installed. So go to your local machine and make sure the environment variable `KUBECONFIG` is not set or points to `~/.kube/config`. Also make sure the `~/.kube/config` file contains no configuration (if you need it, copy it to somewhere else or rename it). The config file should just look like this:
```yaml
apiVersion: v1
clusters: []
contexts: []
current-context: ""
kind: Config
preferences: {}
users: []
```

Now execute the following command (replace the region if necessary):
```sh
aws eks update-kubeconfig --region eu-central-1 --name demo-cluster
# => Added new context arn:aws:eks:eu-central-1:<account-id>:cluster/demo-cluster to ~/.kube/config
```

Now SSH into the DigitalOcean droplet where Jenkins is running:
```sh
ssh root@<jenkins-droplet-ip>
```

Create a file called `config` with the content of the `~/.kube/config` file on your local machine. It looks like this:
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1...S0tCg==
    server: https://7494867E92D8A843B06C6932C8E6D4CD.gr7.eu-central-1.eks.amazonaws.com
  name: arn:aws:eks:eu-central-1:<account-id>:cluster/demo-cluster
contexts:
- context:
    cluster: arn:aws:eks:eu-central-1:<account-id>:cluster/demo-cluster
    user: arn:aws:eks:eu-central-1:<account-id>:cluster/demo-cluster
  name: arn:aws:eks:eu-central-1:<account-id>:cluster/demo-cluster
current-context: arn:aws:eks:eu-central-1:<account-id>:cluster/demo-cluster
kind: Config
preferences: {}
users:
- name: arn:aws:eks:eu-central-1:<account-id>:cluster/demo-cluster
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      args:
      - --region
      - eu-central-1
      - eks
      - get-token
      - --cluster-name
      - demo-cluster
      command: aws
```

In the Jenkins container we don't have aws-cli installed, so we have to replace the command `aws eks get-token --region eu-central-1 --cluster-name demo-cluster` which will be called with every `kubectl` command to authenticate against the AWS account and the EKS cluster with a respective `aws-iam-authenticator token -i demo-cluster` command. So replace the `exec` section in the file with the following:
```yaml
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: aws-iam-authenticator
      args:
        - "token"
        - "-i"
        - "demo-cluster"
```

Now enter the Jenkins container as jenkins user:
```sh
docker exec -it <jenkins-container-id> bash
# => jenkins@<jenkins-container-id>:/$

# navigate to home directory
cd ~
pwd
# /var/jenkins_home

# create a .kube directory
mkdir .kube

# leave the Jenkins container
exit
```

Now copy the created config file into the Jenkins container:
```sh
docker cp config <jenkins-container-id>:/var/jenkins_home/.kube/
```

To change the owner of the file from root to jenkins we enter the Jenkins container as root user:
```sh
docker exec -it -u 0 <jenkins-container-id> bash
chown jenkins:jenkins /var/jenkins_home/.kube/config
```

### Create AWS Credentials
Jenkins needs to know the credentials of the AWS user. In a real project we would create a specific jenkins user on our AWS account with less privileges than an admin user. But for this demo we just add the credentials of the admin user to the Jenkins configuration.

Login to your Jenkins account (running on DigitalOcean) and navigate to Dashboard > devops-bootcamp-multibranch-pipeline > Credentials > devops-bootcamp-multibranch-pipeline > Global credentials (unrestricted) and press "Add Credentials". Select the Kind 'Secret text' and enter 'jenkins-aws_access_key_id' in the 'ID' text field and the aws_access_key_id from your local `~/.aws/credentials` file in the 'Secret' text field. Press the "Create" button. Press "Add Credentials" again and do the same with the aws_secret_access_key.

### Configure Jenkinsfile to deploy to EKS
Go to the sample application 'java-maven-app' and create a new branch 'deploy-on-eks'. Open the Jenkinsfile and replace its content with the following:
```groovy
#!/usr/bin/env groovy

pipeline {
    agent any
    stages {
        stage('build app') {
            steps {
               script {
                   echo "building the application..."
               }
            }
        }
        stage('build image') {
            steps {
                script {
                    echo "building the docker image..."
                }
            }
        }
        stage('deploy') {
            environment {
               AWS_ACCESS_KEY_ID = credentials('jenkins-aws_access_key_id')
               AWS_SECRET_ACCESS_KEY = credentials('jenkins-aws_secret_access_key')
            }
            steps {
                script {
                   echo 'deploying docker image...'
                   sh 'kubectl create deployment nginx-deployment --image=nginx'
                }
            }
        }
    }
}
```

As you can see we just use the 'deploy' stage where we execute a `kubectl` command to create an nginx deployment. If we execute `kubectl` commands on our local machine, authentication against AWS is done using the file `~/.aws/credentials`. Another possibility is to set environment variables `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` containing the same information. As we don't have an `~/.aws/credentials` file in the Jenkins container, we set the two environment variables. The values are taken from the two 'secret text' credentials we just created.

Now when the `kubectl` command is executed, it looks up the kubeconfig file `~/.kube/config` to get the cluster endpoint and cluster authentication data. There it also finds the `aws-iam-authenticator` command to authenticate against AWS. The `aws-iam-authenticator` command needs the AWS credentials (aws-access-key-id and aws-secret-access-key) to do its job.

Note that the aws-iam-authenticator related steps are specific to AWS. On other platforms it would be sufficient to have the kubeconfig file to connect and authenticate with the K8s cluster.

Add, commit and push the new branch to the Git repository.

### Execute Jenkins Pipeline
If we have configured the multibranch pipeline on Jenkins to build all branches, the new branch will be automatically detected, a new pipeline will be created and the build will be automatically started.

After it has successfully finished, open a terminal on your local machine and check the deployment:
```sh
kubectl get pods
# NAME                                READY   STATUS    RESTARTS   AGE
# nginx-deployment-55888b446c-bflcs   1/1     Running   0          2m41s
```

### Links
- [Install kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
- [EKS Cluster Authentication](https://docs.aws.amazon.com/eks/latest/userguide/cluster-auth.html)
- [Create kubeconfig File](https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html)
- [Install AWS IAM Authenticator](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html)

</details>

*****

<details>
<summary>Video: 7 - BONUS: Deploy to LKE Cluster from Jenkins Pipeline</summary>
<br />

In order to deploy to a Linode LKE cluster from a Jenkins pipeline we need to
- make the kubectl command line tool available inside the Jenkins container
- install a Kubernetes CLI Jenkins plugin (to execute kubectl with kubeconfig credentials)
- configure the Jenkinsfile to deploy to the LKE cluster

### Create a Kubernetes Cluster on Linode and Connect to it
Login to your Linode Management Console and navigate to Kubernetes. Press the "Create Cluster" button. Enter a cluster label (e.g. test-cluster), select a region (e.g. Frankfurt eu-central) and the latest K8s version (e.g. 1.26).

Select the "Shared CPU" tab, add one Linode 2 GB node and press "Create Cluster". As soon as the cluster is up and running (this will take just a few moments), download the kubeconfig file `test-cluster-kubeconfig.yaml` that was autogenerated for you.

Right now kubectl is still connecting to the EKS cluster we worked with in the last video (using the default kubeconfig file path `~/.kube/config`). To let kubectl use another config file, just set the environment variable 'KUBECONFIG' to the path of that other config file:
```sh
kubectl get nodes
# NAME                                              STATUS   ROLES    AGE   VERSION
# ip-192-168-48-96.eu-central-1.compute.internal    Ready    <none>   45h   v1.26.2-eks-a59e1f0
# ip-192-168-64-248.eu-central-1.compute.internal   Ready    <none>   45h   v1.26.2-eks-a59e1f0

export KUBECONFIG=~/Downloads/test-cluster-kubeconfig.yaml
kubectl get nodes
# NAME                            STATUS   ROLES    AGE     VERSION
# lke109401-163296-6468ac4fc9f0   Ready    <none>   7m16s   v1.26.3
```

### Add LKE Credentials on Jenkins
Login to your Jenkins account (running on DigitalOcean) and navigate to Dashboard > devops-bootcamp-multibranch-pipeline > Credentials > devops-bootcamp-multibranch-pipeline > Global credentials (unrestricted) and press "Add Credentials". Select the Kind 'Secret file' and upload the `~/Downloads/test-cluster-kubeconfig.yaml` file from your local machine. Enter 'lke-credentials' in the 'ID' text field and press the "Create" button.

### Install Kubernetes CLI Plugin on Jenkins
Navigate to Dashboard > Manage Jenkins > Manage Plugins > Available plugins. Search for "Kubernetes CLI", select the plugin in press "Install without restart".

### Configure Jenkinsfile to Deploy to LKE Cluster
Go to the sample application 'java-maven-app' and create a new branch 'deploy-on-lke'. Open the Jenkinsfile and replace its content with the following:
```groovy
#!/usr/bin/env groovy

pipeline {
    agent any
    stages {
        stage('build app') {
            steps {
               script {
                   echo "building the application..."
               }
            }
        }
        stage('build image') {
            steps {
                script {
                    echo "building the docker image..."
                }
            }
        }
        stage('deploy') {
            steps {
                script {
                   echo 'deploying docker image...'
                   withKubeConfig([credentialsId: 'lke-credentials', serverUrl: '<lke-cluster-endpoint-copied-from-linode>']) {
                       sh 'kubectl create deployment nginx-deployment --image=nginx'
                   }
                }
            }
        }
    }
}
```

As you can see, compared to deploying to AWS EKS cluster, we don't need additional platform authentication configuration. We just use the Kubernetes CLI plugin (`withKubeConfig`) to let the `kubectl` command use the uploaded kubeconfig file.

Add, commit and push the new branch to the Git repository. If we have configured the multibranch pipeline on Jenkins to build all branches, the new branch will be automatically detected, a new pipeline will be created and the build will be automatically started.

After it has successfully finished, open a terminal on your local machine and check the deployment:
```sh
kubectl get pods -o wide
# NAME                                READY   STATUS    RESTARTS   AGE   IP         NODE                            NOMINATED NODE   READINESS GATES
# nginx-deployment-55888b446c-cblwv   1/1     Running   0          10s   10.2.0.6   lke109401-163296-6468ac4fc9f0   <none>           <none>
```

</details>

*****

<details>
<summary>Video: 8 - Jenkins Credentials Note on Best Practices</summary>
<br />

Instead of configuring credentials on Jenkins for our various admin users (e.g. on EC2 instance, EKS cluster, LKE cluster), it is strongly recommended to create dedicated Jenkins users on all of the platforms/environments (e.g. a jenkins service account on AWS), provide these jenkns users only the permissions they need, and configure their credentials on Jenkins server / multi-branch-pipeline.

</details>

*****

<details>
<summary>Video: 9 - Complete CI/CD Pipeline with EKS and DockerHub</summary>
<br />

We would like to replace the deploy stage in the Jenkinsfile of the 'java-maven-app', where we deployed the application on an EC2 instance using `scp` and `ssh` to copy and execute a docker-compose file and a shell script, with our new deploy stage, where we deploy the application to an EKS cluster using `kubcetl`.

So switch to the 'java-maven-app' project and create a new Git branch called 'complete-pipeline-eks-dockerhub'.

### Create Deployment and Service Configuration Files
Create a new folder called `kubernetes` and within this folder two files `deployment.yaml` and `service.yaml` with the following content:

_kubernetes/deployment.yaml_
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: $APP_NAME
  labels:
    app: $APP_NAME
spec:
  replicas: 1
  selector:
    matchLabels:
      app: $APP_NAME
  template:
    metadata:
      labels:
        app: $APP_NAME
    spec:
      imagePullSecrets:
        - name: ...
      containers:
        - name: $APP_NAME
          image: fsiegrist/fesi-repo:devops-bootcamp-java-maven-app-${IMAGE_TAG}
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
```

_kubernetes/service.yaml_
```yaml
apiVersion: v1
kind: Service
metadata:
  name: $APP_NAME
spec:
  selector:
    app: $APP_NAME
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

### Adjust Jenkinsfile
The `IMAGE_TAG` environment variable gets already set in the 'Increment Version' stage of the Jenkinsfile. So we just have to set the new `APP_NAME` variable. We can do it in an `environment` block right inside the 'deploy' stage like this:
```yaml
stage('Deploy Application') {
    environment {
        AWS_ACCESS_KEY_ID = credentials('jenkins-aws_access_key_id')
        AWS_SECRET_ACCESS_KEY = credentials('jenkins-aws_secret_access_key')
        APP_NAME = 'java-maven-app'
    }
    steps {
        script {
            echo 'deploying Docker image to EKS cluster...'
            ...
        }
    }
}
```

We cannot just execute `kubectl apply -f kubernetes/deployment.yaml` (or service.yaml), because these configuration files are actually template files. We first have to substitute the environment variable names with their values. To do this we use the command line tool `envsubst`, which takes a file, looks for env variable references and replaces them with their values. The final commands will then look like this:
```yaml
sh 'envsubst < kubernetes/deployment.yaml | kubectl apply -f -'
sh 'envsubst < kubernetes/service.yaml | kubectl apply -f -'
```

### Install 'gettext-base' command line tool on Jenkins
The `envsubst` command is not available out of the box in the Jenkins container. We have to install it.
```sh
ssh root@<jenkins-droplet-ip>

# enter the container as root to be able to install gettext-base
docker exec -u 0 -it <jenkins-container-id> bash

# install the gettext-base package containing the envsubst tool
apt-get update
apt-get install gettext-base

# check that envsubst is available
which envsubst
# => /usr/bin/envsubst

# leave the Jenkins container
exit

# leave the droplet
exit
```

### Create Secret for DockerHub Credentials
When the deployment configuration file is applied to the EKS cluster, K8s must be able to pull the application image from the private DockerHub registry. For that K8s needs to know the credentials for that private registry. We make these credentials available in a K8s Secret. We could do this inside the Jenkins pipeline (apply a secret.yaml), but because this is a step that has to be executed only once, we can do it from our local machine.

```sh
# make sure we are connecting with the right cluster
kubectl get nodes
# NAME                                              STATUS   ROLES    AGE    VERSION
# ip-192-168-48-96.eu-central-1.compute.internal    Ready    <none>   2d6h   v1.26.2-eks-a59e1f0
# ip-192-168-64-248.eu-central-1.compute.internal   Ready    <none>   2d6h   v1.26.2-eks-a59e1f0

# create the secret
kubectl create secret docker-registry my-registry-key \
  --docker-server=docker.io \
  --docker-username=fsiegrist \
  --docker-password=<password>

kubectl get secrets
# NAME              TYPE                             DATA   AGE
# my-registry-key   kubernetes.io/dockerconfigjson   1      46s
```

Now we have to tell K8s to use this secret when pulling the image. This is done in the `deployment.yaml` file with these two lines added just before `containers:`:
```yaml
imagePullSecrets:
  - name: my-registry-key
```

### Execute Jenkins Pipeline
Add, commit and push the new branch to the Git repository. If we have configured the multibranch pipeline on Jenkins to build all branches, the new branch will be automatically detected, a new pipeline will be created and the build will be automatically started.

After it has successfully finished, open a terminal on your local machine and check the deployment:
```sh
kubectl get pods
# NAME                             READY   STATUS    RESTARTS   AGE
# java-maven-app-f8c8b8d87-zdp48   1/1     Running   0          27s

kubectl describe pod java-maven-app-f8c8b8d87-zdp48
# in the Containers section we see that the image tag was successfully substituted with the updated version

kubectl get deployments
# NAME            READY   UP-TO-DATE   AVAILABLE   AGE
# java-maven-app  1/1     1            1           4m34s

kubectl get services
# NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
# java-maven-app   ClusterIP   10.100.3.216   <none>        80/TCP    5m13s
# kubernetes       ClusterIP   10.100.0.1     <none>        443/TCP   2d7h
```

</details>

*****
