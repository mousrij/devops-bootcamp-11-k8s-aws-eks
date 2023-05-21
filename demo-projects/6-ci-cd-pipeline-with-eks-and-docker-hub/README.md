## Demo Project - Complete CI/CD Pipeline with EKS and private DockerHub registry

### Topics of the Demo Project
Complete CI/CD Pipeline with EKS and private DockerHub registry

### Technologies Used
- Kubernetes
- Jenkins
- AWS EKS
- Docker Hub
- Java
- Maven
- Linux
- Docker
- Git

### Project Description
- Write K8s manifest files for Deployment and Service configuration
- Integrate deploy step in the CI/CD pipeline to deploy newly built application image from DockerHub private registry to the EKS cluster
- So the complete CI/CD project we build has the following configuration:
  - a. CI step: Increment version
  - b. CI step: Build artifact for Java Maven application
  - c. CI step: Build and push Docker image to private DockerHub registry
  - d. CD step: Deploy new application version to EKS cluster
  - e. CD step: Commit the version update

#### Steps to write K8s manifest files for Deployment and Service configuration
We switch to the 'java-maven-app' project and create a new Git branch called 'complete-pipeline-eks-dockerhub'.

We create a new folder called `kubernetes` and within this folder two files `deployment.yaml` and `service.yaml` with the following content:

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

#### Steps to integrate the deploy step in the CI/CD pipeline
**Step 1:** Set the APP_NAME env variable in the Jenkinsfile\
The `IMAGE_TAG` environment variable gets already set in the 'Increment Version' stage of the Jenkinsfile. So we just have to set the new `APP_NAME` variable. We can do it in an `environment` block right inside the 'deploy' stage like this:
```groovy
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

**Step 2:** Add the commands to substitute env variables and apply the configuration files\
We cannot just execute `kubectl apply -f kubernetes/deployment.yaml` (or service.yaml), because these configuration files are actually template files. We first have to substitute the environment variable names with their values. To do this we use the command line tool `envsubst`, which takes a file, looks for env variable references and replaces them with their values. The final commands will then look like this:
```groovy
sh 'envsubst < kubernetes/deployment.yaml | kubectl apply -f -'
sh 'envsubst < kubernetes/service.yaml | kubectl apply -f -'
```

**Step 3:** Install the envsubst tool\
The `envsubst` command is not available out of the box in the Jenkins container. We have to install it.
```sh
ssh root@64.225.104.226

# enter the container as root to be able to install gettext-base
docker exec -u 0 -it 54ae5b80a7c8 bash

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

**Step 4:** Create a K8s Secret containing the DockerHub credentials\
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

**Step 5:** Configure the name of the Secret to be used\
Now we have to tell K8s to use this secret when pulling the image. This is done in the `deployment.yaml` file with these two lines added just before `containers:`:
```yaml
imagePullSecrets:
  - name: my-registry-key
```

**Step 6:** Update the branch name in the git push command\
In the "Commit Version Update" stage we have to adjust the branch in the `git push` command:
```groovy
sh 'git push origin HEAD:complete-pipeline-eks-dockerhub'
```

**Step 7:** Execute the Jenkins pipeline
We add, commit and push the new branch to the Git repository. If we have configured the multibranch pipeline on Jenkins to build all branches, the new branch will be automatically detected, a new pipeline will be created and the build will be automatically started.

After it has successfully finished, we open a terminal on your local machine and check the deployment:
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
