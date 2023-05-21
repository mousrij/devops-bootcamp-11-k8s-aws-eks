## Demo Project - Complete CI/CD Pipeline with EKS and AWS ECR

### Topics of the Demo Project
Complete CI/CD Pipeline with EKS and AWS ECR

### Technologies Used
- Kubernetes
- Jenkins
- AWS EKS
- AWS ECR
- Java
- Maven
- Linux
- Docker
- Git

### Project Description
- Create a private AWS ECR Docker repository
- Create credentials for the ECR repository in Jenkins
- Create a Secret for AWS ECR
- Adjust Jenkinsfile to build and push Docker Image to AWS ECR
- So the complete CI/CD project we build has the following configuration:
  - a. CI step: Increment version
  - b. CI step: Build artifact for Java Maven application
  - c. CI step: Build and push Docker image to AWS ECR
  - d. CD step: Deploy new application version to EKS cluster
  - e. CD step: Commit the version update

#### Steps to create a private AWS ECR Docker repository
In ECR we are not limited to just one private repository (as with DockerHub). So we can create a repository for each application and can use the image tag for the version only.

- Open the browser and login to our account of the [AWS Management Console](https://eu-central-1.console.aws.amazon.com/console/home?region=eu-central-1#).
- Open the ECR dashboard (Services > Containers > Elastic Container Registry) and press the "Get Started" button in the "Create a repository" box.
- Select 'Private' visibility, enter a name (e.g. java-maven-app) and press "Create repository".

#### Steps to create credentials for the ECR repository in Jenkins
First we need the username and password for our ECR repository. Open the ECR dashboard in the AWS Management Console and click on the 'java-maven-app' repository. Press the "View push commands" button. The first command can be used to `docker login` to the ECR:
```sh
aws ecr get-login-password --region eu-central-1 | docker login --username AWS --password-stdin 369076538622.dkr.ecr.eu-central-1.amazonaws.com
```
The first part (`aws ecr get-login-password --region eu-central-1`) retrieves the password which is then passed as stdin to the piped `docker login` command. We also see that the username is 'AWS'.

On our Mac we can copy the password to the clipboard with the following command:
```sh
aws ecr get-login-password --region eu-central-1 | pbcopy
```

- Now we login to our Jenkins Web Console, go to Dashboard > Manage Jenkins > Manage Credentials > System > Global credentials (unrestricted) and press "Add Credentials".
- Select the Kind "Username with password", enter the Username 'AWS', paste the Password copied with the above command and enter the ID 'ecr-credentials'.
- Press "Create".

This is all we need to allow Jenkins to push images to the ECR registry.

#### Steps to create a Secret for AWS ECR
For pulling images from the ECR registry, K8s needs an according Secret holding the same credentials. We can use the command as in the demo project #6 (with updated options):
```sh
kubectl create secret docker-registry aws-registry-key \
  --docker-server=369076538622.dkr.ecr.eu-central-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region eu-central-1)

kubectl get secrets
# NAME               TYPE                             DATA   AGE
# aws-registry-key   kubernetes.io/dockerconfigjson   1      11s
# my-registry-key    kubernetes.io/dockerconfigjson   1      13h
```

#### Steps to adjust Jenkinsfile to build and push Docker Image to AWS ECR
In the "Build and Publish Docker Image" stage we use the repository url several times and it is also hardcoded in the `kubernetes/deployment.yaml` file. It makes sense to extract it to an environment variable. So we add an global environment block before all stages:
```groovy
environment {
    DOCKER_REPO_HOST = '369076538622.dkr.ecr.eu-central-1.amazonaws.com'
    DOCKER_REPO_URI = "${DOCKER_REPO_HOST}/java-maven-app"
}
```

In the "Build and Publish Docker Image" stage we make use of these variables and adjust it like this:
```groovy
stage("Build and Publish Docker Image") {
    steps {
        script {
            withCredentials([usernamePassword(credentialsId: 'ecr-credentials', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                echo "building the docker image..."
                sh "docker build -t ${DOCKER_REPO_URI}:${IMAGE_TAG} ."
                
                echo "publishing the docker image..."
                sh "echo $PASSWORD | docker login -u $USERNAME --password-stdin ${DOCKER_REPO_HOST}"
                sh "docker push ${DOCKER_REPO_URI}:${IMAGE_TAG}"
            }
        }
    }
}
```

In the "Commit Version Update" stage we have to adjust the branch in the `git push` command:
```groovy
sh 'git push origin HEAD:complete-pipeline-eks-ecr'
```

In the `kubernetes/deployment.yaml` file we have to replace the image name:
```yaml
image: ${DOCKER_REPO_URI}:${IMAGE_TAG}
```

Also don't forget to adjust the secret name from `my-registry-key` to `aws-registry-key`.

#### Steps to execute the Jenkins pipeline
We add, commit and push the changes to a new branch called 'complete-pipeline-eks-ecr' of the java-maven-app project. If we have configured the multibranch pipeline on Jenkins to build all branches, the new branch will be automatically detected, a new pipeline will be created and the build will be automatically started.

After it has successfully finished, we open a terminal on our local machine and check the deployment:
```sh
kubectl get pods
# NAME                             READY   STATUS    RESTARTS   AGE
# java-maven-app-f9d749558-fjt64   1/1     Running   0          43s

kubectl describe pod java-maven-app-f9d749558-fjt64
# in the Containers section we see that the image was pulled from the ECR registry

kubectl get deployments
# NAME            READY   UP-TO-DATE   AVAILABLE   AGE
# java-maven-app  1/1     1            1           2m55s

kubectl get services
# NAME             TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
# java-maven-app   ClusterIP   10.100.83.4   <none>        80/TCP    3m12s
# kubernetes       ClusterIP   10.100.0.1    <none>        443/TCP   2d21h
```