pipeline {
    agent any
    environment {
        DOCKER_REGISTRY_NAME = "teds2acr.azurecr.io"
        DOCKER_IMAGE_NAME = "cicd-log4j"
        CS_REGISTRY = "https://hub.docker.com"
        CS_IMAGE_NAME = "mile/cs-fcs"
        CS_IMAGE_TAG = "2.1.0"
        FALCON_REGION = "us-1"
        PROJECT_PATH = "git::https://github.com/mile9299/log4jCICD.git//kubernetes"
        CS_CLIENT_ID = credentials('CS_CLIENT_ID')
        CS_CLIENT_SECRET = credentials('CS_CLIENT_SECRET')
        FCS_CLIENT_ID = credentials('CS_CLIENT_ID')
        FCS_CLIENT_SECRET = credentials('CS_CLIENT_SECRET')
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

        stage('FCS IaC Scan Execution') {
            steps {
               withCredentials([usernameColonPassword(credentialsId: 'CRWD', variable: 'CROWDSTRIKE_CREDENTIALS')]) {
                    script {
                        def SCAN_EXIT_CODE = sh(
                            script: '''
                                set -e
                                echo "=============== FCS IaC Scan Starts ==============="
                                echo "Logging in to DockerHub registry"
                                echo "$CS_PASSWORD" | docker login --username "$CS_USERNAME" --password-stdin
                                
                                echo "Pulling CrowdStrike FCS image"
                                docker pull "$CS_IMAGE_NAME:$CS_IMAGE_TAG" || exit 1
                                
                                echo "Running FCS IaC scan in container"
                                docker run --rm \
                                    -e FCS_CLIENT_ID="$FCS_CLIENT_ID" \
                                    -e FCS_CLIENT_SECRET="$FCS_CLIENT_SECRET" \
                                    "$CS_IMAGE_NAME:$CS_IMAGE_TAG" \
                                    iac scan \
                                    -p "$PROJECT_PATH" \
                                    --falcon-region "$FALCON_REGION" \
                                    --debug
                                
                                SCAN_STATUS=$?
                                echo "Scan completed with status: $SCAN_STATUS"
                                echo "=============== FCS IaC Scan Ends ==============="
                                exit $SCAN_STATUS
                            ''', returnStatus: true
                        )

                        echo "FCS IaC Scan exit code: ${SCAN_EXIT_CODE}"
                        
                        if (SCAN_EXIT_CODE == 40) {
                            echo "FCS IaC Scan found policy violations - marking build as UNSTABLE"
                            currentBuild.result = 'UNSTABLE'
                        } else if (SCAN_EXIT_CODE == 0) {
                            echo "FCS IaC Scan completed successfully"
                            currentBuild.result = 'SUCCESS'
                        } else {
                            echo "FCS IaC Scan failed with unexpected exit code"
                            currentBuild.result = 'FAILURE'
                            error "FCS IaC Scan failed with exit code ${SCAN_EXIT_CODE}"
                        }
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
               // sh 'kubectl create ns preprod-log4j'
                sh 'kubectl apply -f kubernetes/preprod-deployment-log4j.yaml'
            }
        }

        stage('Deploy to Production') {
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: 'Do you want to approve the deployment?', ok: 'Yes'
                }

                echo "Restarting production deployment"
               // sh 'kubectl create ns log4j-prod'
                sh 'kubectl apply -f kubernetes/deployment-log4j.yaml'
            }
        }
    }
}
