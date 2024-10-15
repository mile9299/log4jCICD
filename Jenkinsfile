pipeline {
    agent any
    environment {
        DOCKER_REGISTRY_NAME = "517716713836.dkr.ecr.eu-central-1.amazonaws.com"
        DOCKER_IMAGE_NAME = "cicd-log4j"
        CS_REGISTRY= "registry.crowdstrike.com"
        CS_IMAGE_NAME="registry.crowdstrike.com/fcs/us-1/release/cs-fcs"
        CS_IMAGE_TAG="0.42.0"
        FALCON_REGION="us-1"
        PROJECT_PATH="./kubernetes"
     // DOCKERHUB_CREDENTIALS= credentials('dockerhubcredentials')

        }



    stage('FCS IaC Scan Execution') {
        steps {

            withCredentials([usernamePassword(credentialsId: 'CS_REGISTRY', passwordVariable: 'CS_PASSWORD', usernameVariable: 'CS_USERNAME')]) {
            // Use the credentials here
            echo "Registry username: ${CS_USERNAME}"
            echo "Registry password: ${CS_PASSWORD}"
            }
            withCredentials([usernamePassword(credentialsId: 'CS-API-TOKEN', passwordVariable: 'CS_CLIENT_ID', usernameVariable: 'CS_CLIENT_SECRET')]) {
                            // Use the credentials here
            echo "client ID: ${CS_CLIENT_ID}"
            echo "Client secret: ${CS_CLIENT_SECRET}"   
            }

        }
    }


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

    stage('Image Assessment Crowdstrike') {

            steps {
                withCredentials([usernameColonPassword(credentialsId: 'crwd-talon-1-api-key', variable: '')]) {
                    crowdStrikeSecurity imageName: "${DOCKER_IMAGE_NAME}", imageTag: "${env.BUILD_NUMBER}", enforce: true, timeout: 60
                }
            }
    }

    stage('Push Docker Image to ECR') {
      
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

             steps {              

              sh ("""
                  kubectl rollout restart -f kubernetes/deployment-log4j.yaml
                """)
                
             }
         }
    }
}
