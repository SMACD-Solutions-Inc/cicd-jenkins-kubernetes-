pipeline {
    agent {
        kubernetes {
            yamlFile 'jenkins/pod-template.yaml'
        }
    }
    
    environment {
        APP_NAME = 'my-microservice'
        ECR_REGISTRY = '123456789.dkr.ecr.us-west-2.amazonaws.com'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        SONAR_URL = 'https://sonarqube.company.com'
        K8S_NAMESPACE = 'production'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/company/repo.git',
                    credentialsId: 'github-credentials'
            }
        }
        
        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }
        
        stage('Unit Tests') {
            steps {
                container('maven') {
                    sh 'mvn test'
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                container('maven') {
                    withSonarQubeEnv('SonarQube') {
                        sh """
                            mvn sonar:sonar \
                            -Dsonar.projectKey=${APP_NAME} \
                            -Dsonar.host.url=${SONAR_URL}
                        """
                    }
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                container('docker') {
                    sh """
                        docker build -t ${APP_NAME}:${IMAGE_TAG} .
                        docker tag ${APP_NAME}:${IMAGE_TAG} ${ECR_REGISTRY}/${APP_NAME}:${IMAGE_TAG}
                        docker tag ${APP_NAME}:${IMAGE_TAG} ${ECR_REGISTRY}/${APP_NAME}:latest
                    """
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                container('trivy') {
                    sh """
                        trivy image --severity HIGH,CRITICAL \
                        --exit-code 1 \
                        ${ECR_REGISTRY}/${APP_NAME}:${IMAGE_TAG}
                    """
                }
            }
        }
        
        stage('Push to ECR') {
            steps {
                container('docker') {
                    withAWS(credentials: 'aws-credentials', region: 'us-west-2') {
                        sh """
                            aws ecr get-login-password | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                            docker push ${ECR_REGISTRY}/${APP_NAME}:${IMAGE_TAG}
                            docker push ${ECR_REGISTRY}/${APP_NAME}:latest
                        """
                    }
                }
            }
        }
        
        stage('Update Kubernetes Manifests') {
            steps {
                sh """
                    sed -i 's|IMAGE_TAG|${IMAGE_TAG}|g' k8s/deployment.yaml
                """
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                container('kubectl') {
                    withKubeConfig([credentialsId: 'kubeconfig']) {
                        sh """
                            kubectl apply -f k8s/deployment.yaml -n ${K8S_NAMESPACE}
                            kubectl apply -f k8s/service.yaml -n ${K8S_NAMESPACE}
                            kubectl rollout status deployment/${APP_NAME} -n ${K8S_NAMESPACE}
                        """
                    }
                }
            }
        }
        
        stage('Integration Tests') {
            steps {
                container('maven') {
                    sh 'mvn verify -Pintegration-tests'
                }
            }
        }
    }
    
    post {
        success {
            slackSend(
                color: 'good',
                message: "Pipeline SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}\nImage: ${IMAGE_TAG}"
            )
        }
        failure {
            slackSend(
                color: 'danger',
                message: "Pipeline FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            )
        }
        always {
            cleanWs()
        }
    }
}
