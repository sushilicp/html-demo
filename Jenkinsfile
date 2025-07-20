pipeline {
    agent any
    
    environment {
        DOCKER_HOST = "unix:///var/run/docker.sock"
        CONTAINER_NAME = "my-web-app-${env.BUILD_NUMBER}"
        GOOGLE_CHAT_WEBHOOK = "https://chat.googleapis.com/v1/spaces/AAQAaQR_SNA/messages?key=AIzaSyDdI0hCZtE6vySjMm-WEfRq3CPzqKqqsHI&token=RR8wTSfb0py5U2VnLa53xYIJp2yYxVSWV4wP4ovXPxk"
        DEPLOYMENT_URL = "http://localhost:8080"  // Update this
        DOCKER_HUB_CREDENTIALS = credentials('docker-hub-credentials')
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
                sh """
                    echo ${DOCKER_HUB_CREDENTIALS_PSW} | \
                    docker login -u ${DOCKER_HUB_CREDENTIALS_USR} --password-stdin
                """
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // Login to Docker Hub
                    sh "echo ${DOCKER_HUB_CREDENTIALS_PSW} | docker login -u ${DOCKER_HUB_CREDENTIALS_USR} --password-stdin"
                    
                    // Build multi-stage Docker image
                    sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} -t ${DOCKER_IMAGE}:latest ."
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    // Push both specific tag and latest
                    sh "docker push ${DOCKER_IMAGE}:${DOCKER_TAG}"
                    sh "docker push ${DOCKER_IMAGE}:latest"
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    // Cleanup existing container if needed
                    sh """
                        if docker container inspect ${CONTAINER_NAME} &>/dev/null; then
                            docker stop ${CONTAINER_NAME} || true
                            docker rm ${CONTAINER_NAME} || true
                        fi
                    """
                    
                    // Run new container
                    sh """
                        docker run -d \
                          --name ${CONTAINER_NAME} \
                          -p 8080:80 \
                          ${DOCKER_IMAGE}:${DOCKER_TAG}
                    """
                }
            }
        }
    }

    post {
        always {
            script {
                // Cleanup
                sh """
                    docker logout || true
                    if docker container inspect ${CONTAINER_NAME} &>/dev/null; then
                        docker logs --tail 50 ${CONTAINER_NAME} || true
                        docker stop ${CONTAINER_NAME} || true
                        docker rm ${CONTAINER_NAME} || true
                    fi
                """
                cleanWs()
            }
        }
        success {
            script {
                def message = """
                🚀 *Docker Image Published* 
                *Image*: ${DOCKER_IMAGE}:${DOCKER_TAG}
                *Deployed*: ${CONTAINER_NAME}
                *Access*: http://your-server:8080
                """
                sendGoogleChatNotification(message)
            }
        }
        failure {
            script {
                def logs = sh(script: "docker logs --tail 50 ${CONTAINER_NAME} 2>&1 || true", returnStdout: true)
                def message = """
                🔴 *Deployment Failed* 
                *Build*: #${env.BUILD_NUMBER}
                *Error*: ${currentBuild.currentResult}
                *Logs*:
                ${logs}
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
