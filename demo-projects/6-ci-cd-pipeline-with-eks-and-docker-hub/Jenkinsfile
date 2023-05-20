#!/usr/bin/env groovy

pipeline {
    agent any
    tools {
        maven 'maven-3.9'
    }
    stages {
        stage("Increment Version") {
            steps {
                script {
                    echo 'incrementing the bugfix version of the application...'
                    sh 'mvn build-helper:parse-version versions:set \
                        -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} \
                        versions:commit'
      
                    def version = sh script: 'mvn help:evaluate -Dexpression=project.version -q -DforceStdout', returnStdout: true
                    env.IMAGE_TAG = "$version-$BUILD_NUMBER"
                }
            }
        }
        stage("Build Application JAR") {
            steps {
                script {
                    echo "building the application..."
                    sh 'mvn clean package'
                }
            }
        }
        stage("Build and Publish Docker Image") {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'DockerHub', usernameVariable: 'DOCKER_HUB_USERNAME', passwordVariable: 'DOCKER_HUB_PASSWORD')]) {
                        echo "building the docker image..."
                        sh "docker build -t fsiegrist/fesi-repo:devops-bootcamp-java-maven-app-${IMAGE_TAG} ."
                        
                        echo "publishing the docker image..."
                        sh "echo $DOCKER_HUB_PASSWORD | docker login -u $DOCKER_HUB_USERNAME --password-stdin"
                        sh "docker push fsiegrist/fesi-repo:devops-bootcamp-java-maven-app-${IMAGE_TAG}"
                    }
                }
            }
        }
        stage('Deploy Application') {
            environment {
                AWS_ACCESS_KEY_ID = credentials('jenkins-aws_access_key_id')
                AWS_SECRET_ACCESS_KEY = credentials('jenkins-aws_secret_access_key')
                APP_NAME = 'java-maven-app'
            }
            steps {
                script {
                    echo 'deploying Docker image to EKS cluster...'
                    sh 'envsubst < kubernetes/deployment.yaml | kubectl apply -f -'
                    sh 'envsubst < kubernetes/service.yaml | kubectl apply -f -'
                }
            }
        }
        stage('Commit Version Update') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'GitHub', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                        sh "git remote set-url origin https://${USERNAME}:${PASSWORD}@github.com/fsiegrist/devops-bootcamp-java-maven-app.git"
                        sh 'git add .'
                        sh 'git commit -m "jenkins: version bump"'
                        sh 'git push origin HEAD:main'
                    }
                }
            }
        }
    }   
}
