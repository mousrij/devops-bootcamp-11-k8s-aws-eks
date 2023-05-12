## Notes on the videos for Module 11 "Kubernetes on AQS - EKS"
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

For the worker nodes you need to create and manage EC2 instances (the so called compute fleed) and connect them to the EKS cluster. A semi-managed variant is using EKS with nodegroup, where the EC2 instances are managed for you. All the processes needed on K8s worker nodes (like container runtime, K8s agent, etc.) will be installed on them. But you still have to configure the nodegroups (e.g. scaling behavior). As with ECS it is also possible to combine EKS with Fargate, resulting in fully managed worker nodes.

To create an EKS cluster, you have to 
- provision an EKS cluster (Control Plane Nodes)
- create a Nodegroup of EC2 instances (Worker Nodes)
- connect the Nodegroups to the EKS cluster
- deploy your containerized applications

### Elastic Container Registry (ECR)
ECR is the AWS repository for Docker images (as an alternative to Docker Hub or Nexus). Of course it integrates very well with other AWS services.

</details>

*****

