pipeline {
    agent any
    
    environment {
        DOCKER_HOST = "unix:///var/run/docker.sock"
        DOCKER_IMAGE = "sushilicp/my-web-app"
        DOCKER_TAG = "${env.BUILD_ID ?: 'latest'}"
        CONTAINER_NAME = "my-web-app-${env.BUILD_NUMBER}"
        GOOGLE_CHAT_WEBHOOK = "https://chat.googleapis.com/v1/spaces/AAQAaQR_SNA/messages?key=AIzaSyDdI0hCZtE6vySjMm-WEfRq3CPzqKqqsHI&token=RR8wTSfb0py5U2VnLa53xYIJp2yYxVSWV4wP4ovXPxk"
        DEPLOYMENT_URL = "http://localhost:8080"  // Update this
        DOCKER_HUB_CREDENTIALS = credentials('docker-hub-credentials')
        HOST_PORT = "8080"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', 
                url: 'https://github.com/sushilicp/html-demo.git',
                credentialsId: ''
            }
        }
         stage('Login to Docker Hub') {
            steps {
                script {
                    // Secure way to handle credentials
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-hub-credentials',
                        passwordVariable: 'DOCKER_PASSWORD',
                        usernameVariable: 'DOCKER_USERNAME'
                    )]) {
                        sh """
                            docker login -u ${env.DOCKER_USERNAME} -p ${env.DOCKER_PASSWORD}
                        """
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-hub-credentials',
                        passwordVariable: 'DOCKER_PASSWORD',
                        usernameVariable: 'DOCKER_USERNAME'
                    )]) {
                        sh """
                            docker login -u ${DOCKER_USERNAME} -p ${DOCKER_PASSWORD}
                            docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                        """
                    }
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    sh """
                        docker push ${env.DOCKER_IMAGE}:${env.DOCKER_TAG}
                        docker push ${env.DOCKER_IMAGE}:latest
                    """
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    // Find and stop any container using our port
                    sh """
                        # Find container using port ${HOST_PORT}
                        CONTAINER_USING_PORT=\$(docker ps --format '{{.Names}}' --filter "publish=${HOST_PORT}" | head -n 1)
                        
                        # Stop and remove if found
                        if [ -n "\$CONTAINER_USING_PORT" ]; then
                            echo "Found container \$CONTAINER_USING_PORT using port ${HOST_PORT}, stopping it..."
                            docker stop \$CONTAINER_USING_PORT || true
                            docker rm \$CONTAINER_USING_PORT || true
                        fi
                        
                        # Remove our named container if it exists
                        if docker container inspect ${CONTAINER_NAME} &>/dev/null; then
                            docker stop ${CONTAINER_NAME} || true
                            docker rm ${CONTAINER_NAME} || true
                        fi
                        
                        # Run new container
                        docker run -d \
                            --name ${CONTAINER_NAME} \
                            -p ${HOST_PORT}:80 \
                            ${DOCKER_IMAGE}:${DOCKER_TAG}
                    """
                }
            }
        }
    }

    post {
        always {
            script {
                sh """
                    docker logout || true
                    docker ps -a --filter "name=${CONTAINER_NAME}" --format "{{.ID}}" | xargs -r docker rm -f || true
                """
            }
        }
    
        success {
            script {
                def message = """
                🚀 *Deployment Successful* 
                *Build*: #${env.BUILD_NUMBER}
                *Image*: ${env.DOCKER_IMAGE}:${env.DOCKER_TAG}
                *Container*: ${env.CONTAINER_NAME}
                """
                sendGoogleChatNotification(message)
            }
        }
        failure {
            script {
                def logs = sh(
                    script: "docker logs --tail 50 ${env.CONTAINER_NAME} 2>&1 || true",
                    returnStdout: true
                ).trim()
                
                def message = """
                🔴 *Deployment Failed* 
                *Build*: #${env.BUILD_NUMBER}
                *Error*: ${currentBuild.currentResult}
                *Logs*: ${logs}
                """
                sendGoogleChatNotification(message)
            }
        }
    }
}
def sendGoogleChatNotification(String message) {
    def payload = """
    {
        "text": "${message.replace('"', '\\"').replace('\n', '\\n')}"
    }
    """
    
    sh """
        curl -X POST \
        -H 'Content-Type: application/json' \
        -d '${payload}' \
        '${env.GOOGLE_CHAT_WEBHOOK}' || echo "Notification failed"
    """
}
