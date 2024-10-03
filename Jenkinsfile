pipeline {
    agent any
    environment {
        DOCKER_REGISTRY_NAME = "517716713836.dkr.ecr.eu-central-1.amazonaws.com"
        DOCKER_IMAGE_NAME = "cicd-log4j"
     // DOCKERHUB_CREDENTIALS= credentials('dockerhubcredentials')

        }
    
stages {

    stage('Build WAR') {
            steps {
                dir('vulnerable-application') {
                    sh 'mvn package'
                }
            }
            post {
                success {
                    archiveArtifacts artifacts: 'target/*'
                }
            }
        }
    
    stage('Build Docker Image') {
            when {
                branch 'main'
            }
        steps {
                sh "ls"
                sh "ls vulnerable-application"
                echo 'Building docker image'
                sh "docker build -t ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER} ."
            }
        }

    stage('Push Docker Image to ECR') {
        when {
            branch 'main'
        }
      
        steps {

                echo "Login on ECR"
                sh "aws ecr get-login-password --region eu-central-1 | docker login --username AWS --password-stdin 517716713836.dkr.ecr.eu-central-1.amazonaws.com"       		
	            echo 'Login Completed' 
                echo "Pushing docker image to ECR with current build tag"
                sh " docker tag ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER} ${DOCKER_REGISTRY_NAME}/${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
                sh " docker push ${DOCKER_REGISTRY_NAME}/${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
                echo 'Pushing docker image with tag latest'
                sh "docker tag ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER} ${DOCKER_REGISTRY_NAME}/${DOCKER_IMAGE_NAME}:latest"
                sh "docker push ${DOCKER_REGISTRY_NAME}/${DOCKER_IMAGE_NAME}:latest"
            }
        }
        
        
       stage("Deploy to Production"){
        when {
                branch 'main'
            }
             steps {              

              sh ("""
                  kubectl delete -f deployment-log4j.yaml
                  kubectl apply -f deployment-log4j.yaml
                """)
                
             }
         }
    }
}
