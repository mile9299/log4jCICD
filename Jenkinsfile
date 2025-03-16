pipeline {
    agent any
    environment {
        DOCKER_REGISTRY_NAME = "517716713836.dkr.ecr.eu-central-1.amazonaws.com"
        DOCKER_IMAGE_NAME = "cicd-log4j"
        CS_REGISTRY = "registry.crowdstrike.com"
        CS_IMAGE_NAME = "registry.crowdstrike.com/fcs/us-1/release/cs-fcs"
        CS_IMAGE_TAG = "0.42.0"
        FALCON_REGION = "us-1"
        PROJECT_PATH = "git::https://github.com/grocamador/cicd-log4j.git//kubernetes"
        CS_CLIENT_ID = credentials('CS_CLIENT_ID')
        CS_CLIENT_SECRET = credentials('CS_CLIENT_SECRET')
        CS_USERNAME = credentials('CS_USERNAME')
        CS_PASSWORD = credentials('CS_PASSWORD')
        AWS_CLUSTER_NAME = "your-eks-cluster-name"
        AWS_REGION = "your-aws-region"
    }

    stages {
        stage('Compile') {
            steps {
                dir('vulnerable-application') {
                    sh 'mvn clean package'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Building docker image'
                sh "docker build -t ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER} ."
            }
        }

        stage('Verify AWS CLI & Docker') {
            steps {
                sh 'aws --version'
                sh 'docker --version'
            }
        }

        stage('Authenticate with AWS EKS') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'AWS_CREDENTIALS']]) {
                    sh "aws eks update-kubeconfig --name ${AWS_CLUSTER_NAME} --region ${AWS_REGION}"
                }
            }
        }

        stage('Deploy to Pre') {
            steps {
                withCredentials([file(credentialsId: 'KUBE_CONFIG', variable: 'KUBECONFIG')]) {
                    sh "kubectl rollout restart -f kubernetes/preprod-deployment-log4j.yaml"
                }
            }
        }

        stage('Deploy to Production') {
            steps {
                timeout(time: 15, unit: "MINUTES") {
                    input message: 'Do you want to approve the deployment?', ok: 'Yes'
                }
                withCredentials([file(credentialsId: 'KUBE_CONFIG', variable: 'KUBECONFIG')]) {
                    sh "kubectl rollout restart -f kubernetes/deployment-log4j.yaml"
                }
            }
        }
    }
}
