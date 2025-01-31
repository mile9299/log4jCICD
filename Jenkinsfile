pipeline {
    agent any
    environment {
        DOCKER_REGISTRY_NAME = "517716713836.dkr.ecr.eu-central-1.amazonaws.com"
        DOCKER_IMAGE_NAME = "cicd-log4j"
        CS_REGISTRY= "registry.crowdstrike.com"
        CS_IMAGE_NAME="registry.crowdstrike.com/fcs/us-1/release/cs-fcs"
        CS_IMAGE_TAG="0.42.0"
        FALCON_REGION="us-1"
        PROJECT_PATH="git::https://github.com/grocamador/cicd-log4j.git//kubernetes"
     // DOCKERHUB_CREDENTIALS= credentials('dockerhubcredentials')

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

    stage('Image Assessment Crowdstrike') {

            steps {
                withCredentials([usernameColonPassword(credentialsId: 'crwd-talon-1-api-key', variable: '')]) {
                    crowdStrikeSecurity imageName: "${DOCKER_IMAGE_NAME}", imageTag: "${env.BUILD_NUMBER}", enforce: false, timeout: 60
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
        
    

 stage('FCS IaC Scan Execution') {

    steps {

            withCredentials([usernamePassword(credentialsId: 'CS_REGISTRY', passwordVariable: 'CS_PASSWORD', usernameVariable: 'CS_USERNAME')]) {
            withCredentials([usernamePassword(credentialsId: 'CS-API-TOKEN', passwordVariable: 'CS_CLIENT_SECRET', usernameVariable: 'CS_CLIENT_ID')]) {
    
        script {
            def SCAN_EXIT_CODE = sh(
                script: '''
                    set +x
                    # check if required env vars are set in the build set up

                    scan_status=0
                    if [[ -z "$CS_USERNAME" || -z "$CS_PASSWORD" || -z "$CS_REGISTRY" || -z "$CS_IMAGE_NAME" || -z "$CS_IMAGE_TAG" || -z "$CS_CLIENT_ID" || -z "$CS_CLIENT_SECRET" || -z "$FALCON_REGION" || -z "$PROJECT_PATH" ]]; then
                        echo "Error: required environment variables/params are not set"
                        exit 1
                    else  
                        # login to crowdstrike registry
                        echo "Logging in to crowdstrike registry with username: $CS_USERNAME"
                        echo "$CS_PASSWORD" | docker login "$CS_REGISTRY" --username "$CS_USERNAME" --password-stdin
                        
                        if [ $? -eq 0 ]; then
                            echo "Docker login successful"
                            #  pull the fcs container target
                            echo "Pulling fcs container target from crowdstrike"
                            docker pull "$CS_IMAGE_NAME":"$CS_IMAGE_TAG"
                            if [ $? -eq 0 ]; then
                                echo "fcs docker container image pulled successfully"
                                echo "=============== FCS IaC Scan Starts ==============="

docker run --network=host --rm "$CS_IMAGE_NAME":"$CS_IMAGE_TAG" --client-id "$CS_CLIENT_ID" --client-secret "$CS_CLIENT_SECRET" --falcon-region "$FALCON_REGION" iac scan -p "$PROJECT_PATH" --upload-results --fail-on "high=10,medium=70,low=50,info=10"
                                scan_status=$?
                                echo "=============== FCS IaC Scan Ends ==============="
                            else
                                echo "Error: failed to pull fcs docker image from crowdstrike"
                                scan_status=1
                            fi
                        else
                            echo "Error: docker login failed"
                            scan_status=1
                        fi
                    fi
                ''', returnStatus: true
                )
                echo "fcs-iac-scan-status: ${SCAN_EXIT_CODE}"
                if (SCAN_EXIT_CODE == 40) {
                    echo "Scan succeeded & vulnerabilities count are ABOVE the '--fail-on' threshold; Pipeline will be marked as Success, but this stage will be marked as Unstable"
                    skipPublishingChecks: true
                    currentBuild.result = 'UNSTABLE'
                } else if (SCAN_EXIT_CODE == 0) {
                    echo "Scan succeeded & vulnerabilities count are BELOW the '--fail-on' threshold; Pipeline will be marked as Success"
                    skipPublishingChecks: true
                    skipMarkingBuildUnstable: true
                    currentBuild.result = 'Success'
                } else {
                    currentBuild.result = 'Failure'
                    error 'Unexpected scan exit code: ${SCAN_EXIT_CODE}'
                }
                
        }
            }
            }
    
    }
 }

       stage("Deploy to Pre"){

             steps {              

              sh ("""
                  kubectl rollout restart -f kubernetes/preprod-deployment-log4j.yaml
                """)
                
             }
         }


       stage("Deploy to Production"){

             steps {              
                    // Create an Approval Button with a timeout of 15minutes.
	                timeout(time: 15, unit: "MINUTES") {
	                    input message: 'Do you want to approve the deployment?', ok: 'Yes'
	                }
              sh ("""
                  kubectl rollout restart -f kubernetes/deployment-log4j.yaml
                """)
                
             }
         }
    
    }

}
