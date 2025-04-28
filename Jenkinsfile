pipeline {
    agent any

    tools {
        maven 'maven' // Ensure the Maven installation name matches the one configured in Jenkins
    }

    environment {
        IMAGE_NAME        = "springbootapp"
        IMAGE_TAG         = "${BUILD_NUMBER}" // Use build number as version
        ACR_NAME          = "jenkinsazurejp"
        ACR_LOGIN_SERVER  = "${ACR_NAME}.azurecr.io"
        FULL_IMAGE_NAME   = "${ACR_LOGIN_SERVER}/${IMAGE_NAME}:${IMAGE_TAG}"
        TENANT_ID         = "38fb3b20-a781-4c88-9110-8c817f19c4e1"
        RESOURCE_GROUP    = "JP"
        AKS_CLUSTER       = "springboot"
        K8S_NAMESPACE     = "default"
        K8S_DEPLOYMENT    = "springboot-app"
    }

    stages {
        stage('Checkout From Git') {
            steps {
                git branch: 'prod', url: 'https://github.com/AthiraElza/petclinc-springboot1.git'
            }
        }

        stage('Maven Compile') {
            steps {
                echo "This is Maven Compile Stage"
                sh 'mvn compile'
            }
        }

        stage('Maven Test') {
            steps {
                echo "This is Maven Test Stage"
                sh 'mvn test'
            }
        }

        stage('File System Scan By Trivy') {
            steps {
                echo "Trivy Scan Started"
                sh 'trivy fs --format table --output trivy-report.txt --severity HIGH,CRITICAL .'
            }
        }

        stage('Sonar Analysis') {
            environment {
                SCANNER_HOME = tool 'Sonar-scanner'
            }
            steps {
                withSonarQubeEnv('sonarserver') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.organization=athiraelza \
                        -Dsonar.projectName=petclinc-springboot1 \
                        -Dsonar.projectKey=AthiraElza_petclinc-springboot1 \
                        -Dsonar.java.binaries=. \
                        -Dsonar.exclusions=**/trivy-fs-output.txt
                    '''
                }
            }
        }

        stage('Maven Package') {
            steps {
                echo "Maven Package Started"
                sh 'mvn package'
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    echo "Docker Build Started"
                    docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                }
            }
        }

        stage('Azure Login to ACR') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'azure-acr-sp', usernameVariable: 'AZURE_USERNAME', passwordVariable: 'AZURE_PASSWORD')]) {
                    script {
                        echo "Azure Login Started"
                        sh '''
                            az login --service-principal -u $AZURE_USERNAME -p $AZURE_PASSWORD --tenant $TENANT_ID
                            az acr login --name $ACR_NAME
                        '''
                    }
                }
            }
        }

        stage('Docker Push to ACR') {
            steps {
                script {
                    echo "Docker Push Started"
                    sh '''
                        docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${FULL_IMAGE_NAME}
                        docker push ${FULL_IMAGE_NAME}
                    '''
                }
            }
        }

        stage('Azure Login to Kubernetes') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'azure-acr-sp', usernameVariable: 'AZURE_USERNAME', passwordVariable: 'AZURE_PASSWORD')]) {
                    script {
                        echo "Azure Login to Kubernetes Started"
                        sh '''
                            az login --service-principal -u $AZURE_USERNAME -p $AZURE_PASSWORD --tenant $TENANT_ID
                            az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER --overwrite-existing    
                        '''
                    }
                }
            }
        }
    }
}
