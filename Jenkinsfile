pipeline {
    agent {
        label 'QA'
    }

    parameters {
        string(
            name: 'Branch',
            defaultValue: 'main',
            description: 'Branch Name'
        )
    }

    environment {
        ACR_LOGIN_SERVER = 'registryplatform.powermindinc.com'
        REPOSITORY_NAME = 'test-service-app'
        IMAGE_TAG = "${BUILD_NUMBER}"
        IMAGE_NAME = "registryplatform.powermindinc.com/test-service-app:${BUILD_NUMBER}"
        NAMESPACE = ''
    }

    stages {

        stage('Build Image') {
            steps {
                script {
                    echo "Building Docker image: ${IMAGE_NAME}"

                    sh """
                        docker build \
                            -t ${IMAGE_NAME} \
                            .
                    """
                }
            }
        }

        stage('Login to Registry') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'docker-registry-cred',
                            usernameVariable: 'SERVICE_PRINCIPAL_ID',
                            passwordVariable: 'SERVICE_PRINCIPAL_PASSWORD'
                        )
                    ]) {
                        sh '''
                            echo "$SERVICE_PRINCIPAL_PASSWORD" | docker login \
                                "$ACR_LOGIN_SERVER" \
                                -u "$SERVICE_PRINCIPAL_ID" \
                                --password-stdin
                        '''
                    }
                }
            }
        }

        stage('Push Image') {
            steps {
                script {
                    echo "Pushing image: ${IMAGE_NAME}"

                    sh """
                        docker push ${IMAGE_NAME}
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    withKubeConfig(
                        credentialsId: 'k8s-vibecode-env',
                        serverUrl: 'https://172.21.200.15:6443'
                    ) {
                        sh """
                            kubectl apply \
                                -f test.yaml \
                                -n ${NAMESPACE}

                            kubectl set image deployment/test-service-app \
                                test-service-app=${IMAGE_NAME} \
                                -n ${NAMESPACE}

                            kubectl rollout status deployment/test-service-app \
                                -n ${NAMESPACE} \
                                --timeout=180s
                        """
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    withKubeConfig(
                        credentialsId: 'k8s-vibecode-env',
                        serverUrl: 'https://172.21.200.15:6443'
                    ) {
                        sh """
                            echo "Deployment:"
                            kubectl get deployment test-service-app -n ${NAMESPACE}

                            echo "Pods:"
                            kubectl get pods \
                                -l app=test-service-app \
                                -n ${NAMESPACE}

                            echo "Service:"
                            kubectl get service test-service-app -n ${NAMESPACE}
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "=========================================="
            echo "Deployment Successful"
            echo "Image: ${IMAGE_NAME}"
            echo "Namespace: ${NAMESPACE}"
            echo "=========================================="
        }

        failure {
            echo "=========================================="
            echo "Deployment Failed"
            echo "=========================================="
        }

        always {
            sh 'docker logout "$ACR_LOGIN_SERVER" || true'
        }
    }
}
