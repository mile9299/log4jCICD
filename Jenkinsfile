pipeline {
    agent any
    environment {
        DOCKER_REGISTRY_NAME = "teds2acr.azurecr.io"
        DOCKER_IMAGE_NAME = "cicd-log4j"
        CS_REGISTRY = "https://hub.docker.com"
        CS_IMAGE_NAME = "mile/cs-fcs"
        CS_IMAGE_TAG = "1.0.0"
        FALCON_REGION = "us-1"
        PROJECT_PATH = "git::https://github.com/mile9299/log4jCICD.git//kubernetes"
        CS_CLIENT_ID = credentials('CS_CLIENT_ID')
        CS_CLIENT_SECRET = credentials('CS_CLIENT_SECRET')
        CS_USERNAME = 'mile'
        CS_PASSWORD = credentials('CS_PASSWORD')
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
                sh 'docker build -t $DOCKER_IMAGE_NAME:$BUILD_NUMBER .'
            }
        }

        stage('Verify AZ CLI & Docker') {
            steps {
                sh 'az --version'
                sh 'docker --version'
            }
        }

        stage('Test with Snyk') {
            steps {
                dir('vulnerable-application') {
                    script {
                        snykSecurity failOnIssues: false, severity: 'critical', snykInstallation: 'snyk-manual', snykTokenId: 'SNYK'
                    }
                }
            }
        }

        stage('Image Assessment CrowdStrike') {
            steps {
                withCredentials([usernameColonPassword(credentialsId: 'CRWD', variable: 'CROWDSTRIKE_CREDENTIALS')]) {
                    crowdStrikeSecurity imageName: "$DOCKER_IMAGE_NAME", imageTag: "$BUILD_NUMBER", enforce: false, timeout: 60
                }
            }
        }

        stage('Push Docker Image to ACR') {
            steps {
                echo "Logging into ACR securely"
                withCredentials([usernamePassword(credentialsId: 'ACR_CREDENTIALS', usernameVariable: 'ACR_USER', passwordVariable: 'ACR_PASSWORD')]) {
                    sh 'docker login teds2acr.azurecr.io -u $ACR_USER -p $ACR_PASSWORD'
                }

                echo 'Pushing docker image to ACR with build tag'
                sh 'docker tag $DOCKER_IMAGE_NAME:$BUILD_NUMBER teds2acr.azurecr.io/$DOCKER_IMAGE_NAME:$BUILD_NUMBER'
                sh 'docker push teds2acr.azurecr.io/$DOCKER_IMAGE_NAME:$BUILD_NUMBER'

                echo 'Pushing docker image with tag latest'
                sh 'docker tag $DOCKER_IMAGE_NAME:$BUILD_NUMBER teds2acr.azurecr.io/$DOCKER_IMAGE_NAME:latest'
                sh 'docker push teds2acr.azurecr.io/$DOCKER_IMAGE_NAME:latest'
            }
        }

        stage('Falcon Cloud Security IaC Scan') {
            steps {
                script {
                    def SCAN_EXIT_CODE = sh(
                        script: '''
                            set +x
                            scan_status=0
                            if [[ -z "$CS_USERNAME" || -z "$CS_PASSWORD" || -z "$CS_REGISTRY" || -z "$CS_IMAGE_NAME" || -z "$CS_IMAGE_TAG" || -z "$CS_CLIENT_ID" || -z "$CS_CLIENT_SECRET" || -z "$FALCON_REGION" || -z "$PROJECT_PATH" ]]; then
                                echo "Error: required environment variables/params are not set"
                                exit 1
                            fi
                            
                            echo "Logging in to crowdstrike registry with username: $CS_USERNAME"
                            echo "$CS_PASSWORD" | docker login --username "$CS_USERNAME" --password-stdin
                            
                            echo "Pulling fcs container target from crowdstrike"
                            docker pull mile/cs-fcs:1.0.0
                            
                            echo "=============== FCS IaC Scan Starts ==============="
                            docker run --network=host --rm "$CS_IMAGE_NAME":"$CS_IMAGE_TAG" \
                                --client-id "$CS_CLIENT_ID" \
                                --client-secret "$CS_CLIENT_SECRET" \
                                --falcon-region "$FALCON_REGION" \
                                iac scan -p "$PROJECT_PATH"
                            scan_status=$?
                            echo "=============== FCS IaC Scan Ends ==============="
                            exit $scan_status
                        ''',
                        returnStatus: true
                    )
                    
                    echo "fcs-iac-scan-status: ${SCAN_EXIT_CODE}"
                    if (SCAN_EXIT_CODE == 40) {
                        echo "Scan succeeded & vulnerabilities count are ABOVE the '--fail-on' threshold"
                        currentBuild.result = 'UNSTABLE'
                    } else if (SCAN_EXIT_CODE == 0) {
                        echo "Scan succeeded & vulnerabilities count are BELOW the '--fail-on' threshold"
                        currentBuild.result = 'SUCCESS'
                    } else {
                        error "Unexpected scan exit code: ${SCAN_EXIT_CODE}"
                    }
                }
            }
        }

        stage('Deploy to Pre') {
            steps {
                echo "Logging into Azure securely"
                withCredentials([ 
                    string(credentialsId: 'AZURE_SUBSCRIPTION', variable: 'AZ_SUB_ID'),
                    string(credentialsId: 'AZURE_TENANT_ID', variable: 'AZ_TENANT_ID'),
                    usernamePassword(credentialsId: 'AZURE_SERVICE_PRINCIPAL', usernameVariable: 'AZURE_SP_ID', passwordVariable: 'AZURE_SP_SECRET')
                ]) {
                    sh '''
                        az login --service-principal -u $AZURE_SP_ID -p $AZURE_SP_SECRET --tenant $AZ_TENANT_ID
                        if [ $? -ne 0 ]; then
                            echo "Azure login failed. Please check your credentials or permissions."
                            exit 1
                        fi
                    '''
                }

                echo "Fetching AKS credentials"
                sh '''
                    az aks get-credentials --resource-group TedsAKS_group --name TedsAKS --overwrite-existing
                    if [ $? -ne 0 ]; then
                        echo "Failed to fetch AKS credentials. Please check your permissions."
                        exit 1
                    fi
                '''

                echo "Deploying to preprod"
                sh 'kubectl apply -f kubernetes/preprod-deployment-log4j.yaml'
            }
        }

        stage('Deploy to Production') {
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: 'Do you want to approve the deployment?', ok: 'Yes'
                }

                echo "Restarting production deployment"
                sh 'kubectl apply -f kubernetes/deployment-log4j.yaml'
            }
        }
    }

    post {
        success {
            echo 'Build succeeded!'
        }
        unstable {
            echo 'Build is unstable, but still considered successful!'
        }
        failure {
            echo 'Build failed!'
        }
        always {
            echo "FCS IaC Scan Execution complete.."
        }
    }
}
