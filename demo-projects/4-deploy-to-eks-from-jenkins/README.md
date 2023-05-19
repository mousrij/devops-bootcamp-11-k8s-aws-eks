## Demo Project - Deploy to EKS cluster from Jenkins Pipeline

### Topics of the Demo Project
CD - Deploy to EKS cluster from Jenkins Pipeline

### Technologies Used
- Kubernetes
- Jenkins
- AWS EKS
- Docker
- Linux

### Project Description
- Install kubectl and aws-iam-authenticator on a Jenkins server
- Create kubeconfig file to connect to EKS cluster and add it on Jenkins server
- Add AWS credentials on Jenkins for AWS account authentication
- Extend and adjust Jenkinsfile of the previous CI/CD pipeline to configure connection to EKS cluster

#### Steps to install kubectl and aws-iam-authenticator on a Jenkins server
SSH into the DigitalOcean droplet where Jenkins is running:
```sh
ssh root@64.225.104.226

# make sure jenkins is running
docker ps
# CONTAINER ID   IMAGE                 COMMAND                  CREATED        STATUS         PORTS                                                                                      NAMES
# 54ae5b80a7c8   jenkins/jenkins:lts   "/usr/bin/tini -- /u…"   2 months ago   Up 3 minutes   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp, 0.0.0.0:50000->50000/tcp, :::50000->50000/tcp   nervous_euler

# enter the container as root to be able to install the required tools
docker exec -u 0 -it 54ae5b80a7c8 bash

# ---- install kubectl ----

# get the current commands to install kubectl from https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

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

# ---- install aws-iam-authenticator ----

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

#### Steps to create kubeconfig file to connect to EKS cluster and add it on Jenkins server
Inside the Jenkins Docker container we don't have an editor available, so we create the kubeconfig file outside the container (on the DigitalOcean droplet) and copy it into the container.

But to get the content of the file, we create it on our local machine where aws-cli is installed. On our local machine we make sure the environment variable `KUBECONFIG` is not set or points to `~/.kube/config`. We also make sure the `~/.kube/config` file contains no configuration. It should just look like this:
```yaml
apiVersion: v1
clusters: []
contexts: []
current-context: ""
kind: Config
preferences: {}
users: []
```

Now we execute the following command:
```sh
aws eks update-kubeconfig --region eu-central-1 --name demo-cluster
# => Added new context arn:aws:eks:eu-central-1:369076538622:cluster/demo-cluster to /Users/fsiegrist/.kube/config
```

Now we ssh into the DigitalOcean droplet where Jenkins is running...
```sh
ssh root@64.225.104.226
```

...and create a file called `config` with the content of the `~/.kube/config` file on our local machine. It looks like this:
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1...S0tCg==
    server: https://7494867E92D8A843B06C6932C8E6D4CD.gr7.eu-central-1.eks.amazonaws.com
  name: arn:aws:eks:eu-central-1:369076538622:cluster/demo-cluster
contexts:
- context:
    cluster: arn:aws:eks:eu-central-1:369076538622:cluster/demo-cluster
    user: arn:aws:eks:eu-central-1:369076538622:cluster/demo-cluster
  name: arn:aws:eks:eu-central-1:369076538622:cluster/demo-cluster
current-context: arn:aws:eks:eu-central-1:369076538622:cluster/demo-cluster
kind: Config
preferences: {}
users:
- name: arn:aws:eks:eu-central-1:369076538622:cluster/demo-cluster
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

In the Jenkins container we don't have aws-cli installed, so we have to replace the command `aws eks get-token --region eu-central-1 --cluster-name demo-cluster`, which will be called with every `kubectl` command to authenticate against the AWS account and the EKS cluster with a respective `aws-iam-authenticator token -i demo-cluster` command. So we replace the `exec` section in the file with the following:
```yaml
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: aws-iam-authenticator
      args:
        - "token"
        - "-i"
        - "demo-cluster"
```

Now we enter the Jenkins container as jenkins user:
```sh
docker exec -it 54ae5b80a7c8 bash
# => jenkins@54ae5b80a7c8:/$

# navigate to home directory
cd ~
pwd
# /var/jenkins_home

# create a .kube directory
mkdir .kube

# leave the Jenkins container
exit
```

Now we copy the created config file into the Jenkins container:
```sh
docker cp config 54ae5b80a7c8:/var/jenkins_home/.kube/
```

To change the owner of the file from root to jenkins we enter the Jenkins container as root user:
```sh
docker exec -it -u 0 54ae5b80a7c8 bash
chown jenkins:jenkins /var/jenkins_home/.kube/config
```

#### Steps to add AWS credentials on Jenkins for AWS account authentication
Jenkins needs to know the credentials of the AWS user. In a real project we would create a specific jenkins user on our AWS account with less privileges than an admin user. But for this demo we just add the credentials of the admin user to the Jenkins configuration.

We login to our [Jenkins account](http://64.225.104.226:8080), navigate to Dashboard > devops-bootcamp-multibranch-pipeline > Credentials > devops-bootcamp-multibranch-pipeline > Global credentials (unrestricted) and press "Add Credentials". We select the Kind 'Secret text' and enter 'jenkins-aws_access_key_id' in the 'ID' text field and the aws_access_key_id from our local `~/.aws/credentials` file in the 'Secret' text field. We press the "Create" button. Now we press "Add Credentials" again and do the same with the aws_secret_access_key.

#### Steps to extend and adjust Jenkinsfile of the previous CI/CD pipeline to configure connection to EKS cluster
We switch to the sample application 'java-maven-app' and create a new branch 'deploy-on-eks'. We replace the content of the Jenkinsfile with the following:
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

We just use the 'deploy' stage where we execute a `kubectl` command to create an nginx deployment. If we execute `kubectl` commands on our local machine, authentication against AWS is done using the file `~/.aws/credentials`. Another possibility is to set environment variables `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` containing the same information. As we don't have an `~/.aws/credentials` file in the Jenkins container, we set the two environment variables. The values are taken from the two 'secret text' credentials we just created.

We add, commit and push the new branch to the Git repository.

### Execute Jenkins Pipeline
If we have configured the multibranch pipeline on Jenkins to build all branches, the new branch will be automatically detected, a new pipeline will be created and the build will be automatically started.

After it has successfully finished, we open a terminal on our local machine and check the deployment:
```sh
kubectl get pods
# NAME                                READY   STATUS    RESTARTS   AGE
# nginx-deployment-55888b446c-bflcs   1/1     Running   0          2m41s
```
