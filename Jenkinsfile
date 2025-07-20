pipeline {
    agent any
    
    environment {
        DOCKER_HOST = "unix:///var/run/docker.sock"
        DOCKER_IMAGE = "sushilicp/my-web-app"
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
                    // Build with proper tagging
                    sh """
                        docker build -t ${env.DOCKER_IMAGE}:${env.DOCKER_TAG} .
                        docker tag ${env.DOCKER_IMAGE}:${env.DOCKER_TAG} ${env.DOCKER_IMAGE}:latest
                    """
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
                    // Cleanup existing container if it exists
                    sh """
                        if docker container inspect ${env.CONTAINER_NAME} &>/dev/null; then
                            docker stop ${env.CONTAINER_NAME} || true
                            docker rm ${env.CONTAINER_NAME} || true
                        fi
                        
                        docker run -d \
                          --name ${env.CONTAINER_NAME} \
                          -p 8080:80 \
                          ${env.DOCKER_IMAGE}:${env.DOCKER_TAG}
                    """
                }
            }
        }
    }

    post {
        always {
            script {
                // Basic cleanup without cleanWs
                sh """
                    docker logout || true
                    rm -rf * || true
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
