## Exercises for Module 11 "Kubernetes on AWS - EKS"
<br />

Right after you setup the cluster on LKE or Minikube and deployed your application inside, your manager comes to you to say, that the company wants to run Kubernetes also on AWS, again less overhead when managing just one platform. So they ask you to reconfigure your cluster on AWS and deploy your application there instead.

<details>
<summary>Exercise 1: Create an EKS cluster</summary>
<br />

**Tasks:**

You decide to create an EKS cluster - the managed Kubernetes Service of AWS. To simplify the whole creation and configurations, you use eksctl.
- With eksctl you create an EKS cluster with 3 Nodes and 1 Fargate profile

**Steps to solve the tasks:**

**Step 1:** Install eksctl\
On a Mac with M2 processor we could install eksctl using homebrew:
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

**Step 2:** Configure AWS credentials to connect eksctl with our AWS account
Because we already have configured credentials for the `aws` CLI tool we can use the same configuration for `eksctl`. Otherwhise we would have to tell `eksctl` with which account and which user we want to connect. Using the file that was downloaded when the access key for the admin user was created we would execute the following command:

```sh
aws configure
  AWS Access Key ID [None]: # enter the AWS access key id from the downloaded .csv file
  AWS Secret Access Key [None]: # enter the AWS secret access key from the downloaded .csv file
  Default region name [None]: eu-central-1 # Frankfurt
  Default output format [None]: json
```

This configuration will be used for all subsequent `eksctl` (or `aws`) commands. The configuration itself is stored in `~/.aws/config` and `~/.aws/credentials`.

**Step 3:** Create a cluster with 3 EC2 instances
```sh
# create cluster and store access configuration in kubeconfig.exercise-cluster.yaml file 
eksctl create cluster \
  --name=exercise-cluster \
  --version 1.26 \
  --region eu-central-1 \
  --nodes=3 \
  --kubeconfig=./kubeconfig.exercise-cluster.yaml

# 2023-05-21 22:05:12 [ℹ]  eksctl version 0.141.0
# 2023-05-21 22:05:12 [ℹ]  using region eu-central-1
# ...
# 2023-05-21 22:21:18 [ℹ]  kubectl command should work with "./kubeconfig.exercise-cluster.yaml", try 'kubectl --kubeconfig=./kubeconfig.exercise-cluster.yaml get nodes'
# 2023-05-21 22:21:18 [✔]  EKS cluster "exercise-cluster" in "eu-central-1" region is ready
```

**Step 4:** Create a Fargate profile in the cluster
```sh
eksctl create fargateprofile \
  --cluster exercise-cluster \
  --name exercise-fargate-profile \
  --namespace ns-exercise

# 2023-05-21 22:28:58 [ℹ]  deploying stack "eksctl-exercise-cluster-fargate"
# ...
# 2023-05-21 22:29:29 [ℹ]  creating Fargate profile "exercise-fargate-profile" on EKS cluster "exercise-cluster"
# 2023-05-21 22:31:39 [ℹ]  created Fargate profile "exercise-fargate-profile" on EKS cluster "exercise-cluster"
```

**Step 5:** Point kubectl to your cluster
```sh
export KUBECONFIG=$(pwd)/kubeconfig.exercise-cluster.yaml
```

**Step 6:** Validate cluster is accessible and nodes and Fargate profile created
```sh
kubectl get nodes
# NAME                                              STATUS   ROLES    AGE     VERSION
# ip-192-168-13-226.eu-central-1.compute.internal   Ready    <none>   7m56s   v1.26.4-eks-0a21954
# ip-192-168-37-194.eu-central-1.compute.internal   Ready    <none>   7m56s   v1.26.4-eks-0a21954
# ip-192-168-79-85.eu-central-1.compute.internal    Ready    <none>   7m56s   v1.26.4-eks-0a21954

eksctl get fargateprofile --cluster exercise-cluster
# NAME				SELECTOR_NAMESPACE	SELECTOR_LABELS	POD_EXECUTION_ROLE_ARN									SUBNETS										TAGS	STATUS
# exercise-fargate-profile	ns-exercise		<none>		arn:aws:iam::369076538622:role/eksctl-exercise-cluster-fa-FargatePodExecutionRole-BSX2AXPC48R5	subnet-0a441f86ecee59c28,subnet-0bba36a8601a77e4d,subnet-0be946357dafe88ee	<none>	ACTIVE
``` 

</details>

******

<details>
<summary>Exercise 2: Deploy Mysql and phpmyadmin</summary>
<br />

**Tasks:**
- You deploy mysql and phpmyadmin on EC2 nodes with the same setup as before.

**Steps to solve the tasks:**

**Step 1:** Install helm\
If we hadn't installed Helm yet, we would [install it now](https://helm.sh/docs/intro/install/). On a Mac, the easiest way to install Helm would be to execute
```sh
brew update
brew install helm
```

**Step 2:** Get and configure Helm Chart for MySql\
Google for "Helm Charts Mysql". You should find the charts maintained by [Bitnami](https://bitnami.com/stack/mysql/helm). Execute the following commands:
```sh
# add the bitnami repo
helm repo add bitnami https://charts.bitnami.com/bitnami

# search for mysql charts in this repo
helm search repo bitnami/mysql
# =>
# NAME            CHART VERSION	  APP VERSION   DESCRIPTION                                       
# bitnami/mysql   9.8.2           8.0.33     	MySQL is a fast, reliable, scalable, and easy t...
```

To see the parameters of the chart, open the browser and navigate to `https://github.com/bitnami/charts/tree/main/bitnami/mysql`. You'll find that there are parameters `architecture`, `auth.rootPassword`, `secondary.replicaCount`, `secondary.persistence.storageClass` (among many others). To override these parameters for deployment on an EKS cluster create a file called `mysql-chart-values-eks.yaml` in the `k8s` folder with the following content:
```yaml
architecture: replication
auth:
  rootPassword: secret-root-pass
  database: my-app-db
  username: my-user
  password: my-pass

# enable init container that changes the owner and group of the persistent volume mountpoint to runAsUser:fsGroup
volumePermissions:
  enabled: true

primary:
  persistence:
    enabled: false

secondary:
  # 1 primary and 2 secondary replicas
  replicaCount: 2
  persistence:
    enabled: false
```

To install the chart in the EKS cluster execute the following commands:
```sh
helm install -f k8s/mysql-chart-values-eks.yaml my-release bitnami/mysql

kubectl get all
# NAME                               READY   STATUS    RESTARTS   AGE
# pod/my-release-mysql-primary-0     1/1     Running   0          86s
# pod/my-release-mysql-secondary-0   1/1     Running   0          86s
# pod/my-release-mysql-secondary-1   1/1     Running   0          36s
# 
# NAME                                          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
# service/kubernetes                            ClusterIP   10.100.0.1      <none>        443/TCP    26m
# service/my-release-mysql-primary              ClusterIP   10.100.77.54    <none>        3306/TCP   86s
# service/my-release-mysql-primary-headless     ClusterIP   None            <none>        3306/TCP   86s
# service/my-release-mysql-secondary            ClusterIP   10.100.253.98   <none>        3306/TCP   86s
# service/my-release-mysql-secondary-headless   ClusterIP   None            <none>        3306/TCP   86s
# 
# NAME                                          READY   AGE
# statefulset.apps/my-release-mysql-primary     1/1     86s
# statefulset.apps/my-release-mysql-secondary   2/2     86s
```

**Step 3:** Install phpmyadmin
```sh
# deploy phpmyadmin with its configuration for Mysql DB access
kubectl apply -f k8s/db-config.yaml
kubectl apply -f k8s/db-secret.yaml
kubectl apply -f k8s/phpmyadmin.yaml
```

**Step 4:** Access phpmyadmin
```sh
# access phpmyadmin and login to mysql db
kubectl port-forward svc/phpmyadmin-service 8081:8081
```

Open the browser and navigate to [localhost:8081](http://localhost:8081). Login with either "my-user:my-pass" or "root:secret-root-pass".

</details>

******

<details>
<summary>Exercise 3: Deploy your Java application</summary>
<br />

**Tasks:**
- You deploy your Java application using Fargate with 3 replicas and same setup as before

**Steps to solve the tasks:**

**Step 1:** Create namespace ns-exercise\
Because we want to deploy the java-app with Fargate profile and Fargate profile we created applies for ns-exercise namespace we have to create this namespace: 
```sh
kubectl create namespace ns-exercise
```

Now we have to create all configuration and secrets for our java app in the ns-exercise namespace.

**Step 2:** Create my-registry-key Secret to pull image
```sh
kubectl create secret -n ns-exercise docker-registry my-registry-key \
  --docker-server=docker.io \
  --docker-username=fsiegrist \
  --docker-password=<password>
```

**Step 3:** Push docker image of java mysql app to private registry if necessary\
Go to the [bootcamp-java-mysql](https://github.com/fsiegrist/devops-bootcamp-07-docker/tree/main/bootcamp-java-mysql) app from the exercises of module 7. Set the version in `build.gradle` to '1.3-SNAPSHOT', adjust the versions in the `Dockerfile` accordingly and make sure, host and port in `src/main/resources/static/index.html` is set to '???'.

Build the jar file executing
```sh
./gradlew build
```

Create a docker image executing 
```sh
docker build -t fsiegrist/fesi-repo:bootcamp-java-mysql-project-1.2-SNAPSHOT .
```

Push the image to remote private registry on DockerHub executing
```sh
docker login
docker push fsiegrist/fesi-repo:bootcamp-java-mysql-project-1.2-SNAPSHOT
```

**Step 4:** Deploy java-mysql-app in ns-exercise namespace
```sh
kubectl apply -f k8s/db-secret.yaml -n ns-exercise
kubectl apply -f k8s/db-config.yaml -n ns-exercise
kubectl apply -f k8s/java-mysql-app.yaml -n ns-exercise
```

</details>

******

### Setup Continuous Deployment with Jenkins

<details>
<summary>Exercise 4: Automate deployment</summary>
<br />

**Tasks:**

Now your application is running. And when you or others make changes to it, Jenkins pipeline builds the new image, but you have to manually deploy it into the cluster. But you know how annoying that is for you and your team from experience, so you want to automate deploying to the cluster as well.
- Setup automatic deploying to the cluster in the pipeline.


**Steps to solve the tasks:**


</details>

******

<details>
<summary>Exercise 5: Use ECR as Docker repository</summary>
<br />

**Tasks:**

Now your manager comes and tells you that all the projects in the company have containerized their applications, so there is no need for keeping and managing Nexus on the Droplet. Also since all projects have CI/CD there are hundreds of images pushed per day to Nexus repository and you need to manage the storage and cleanup policies to make space.

So, company wants to use ECR instead, again to have everything on 1 platform and also to let AWS manage the repository with storage, cleanups etc.
- Therefore, you need to replace the docker repository in your pipeline with ECR


**Steps to solve the tasks:**


</details>

******

<details>
<summary>Exercise 6: Configure Autoscaling</summary>
<br />

**Tasks:**

Now your application is running, whenever a change is made, it gets automatically deployed in the cluster etc. This is great, but you notice that most of the time the 3 nodes you have are underutilized, especially at the weekends, because your containers aren't using that much resources. However, your company is paying the full price for all the servers.

So you suggest to your manager, that you will be able to save the company some infrastructure costs, by configuring autoscaling. Your manager is happy about that and asks you to configure it.
- So go ahead and configure autoscaling to scale down to minimum 1 node when servers are underutilized and maximum 3 nodes when in full use.


**Steps to solve the tasks:**


</details>

******