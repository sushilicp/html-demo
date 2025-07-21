pipeline {
    agent any
    
    environment {
        DOCKER_HOST = "npipe:////./pipe/docker_engine" // Windows Docker endpoint
        DOCKER_IMAGE = "sushilicp/my-web-app"
        DOCKER_TAG = "${env.BUILD_ID ?: 'latest'}"
        CONTAINER_NAME = "my-web-app-${env.BUILD_NUMBER}"
        GOOGLE_CHAT_WEBHOOK = credentials('google-chat-webhook') // Secure webhook
        DEPLOYMENT_URL = "http://localhost:8088"
        DOCKER_HUB_CREDENTIALS = credentials('docker-hub-credentials')
        HOST_PORT = "8088"
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
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-hub-credentials',
                        passwordVariable: 'DOCKER_PASSWORD',
                        usernameVariable: 'DOCKER_USERNAME'
                    )]) {
                        bat """
                            docker login -u %DOCKER_USERNAME% -p %DOCKER_PASSWORD%
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
                        bat """
                            docker login -u %DOCKER_USERNAME% -p %DOCKER_PASSWORD%
                            docker build -t %DOCKER_IMAGE%:%DOCKER_TAG% .
                        """
                    }
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    bat """
                        docker push %DOCKER_IMAGE%:%DOCKER_TAG%
                        docker push %DOCKER_IMAGE%:latest
                    """
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    // Use sh for complex shell script, requires Git Bash or WSL
                    bat """
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
            node('built-in') {
                script {
                    // Use bat for simple Docker commands
                    bat """
                        docker logout || exit 0
                        docker ps -a --filter "name=%CONTAINER_NAME%" --format "{{.ID}}" | for /f %%i in ('more') do docker rm -f %%i || exit 0
                    """
                }
            }
        }
    
        success {
            node('built-in') {
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
        }
        failure {
            node('built-in') {
                script {
                    def logs = bat(
                        script: "docker logs --tail 50 %CONTAINER_NAME% 2>&1 || exit 0",
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
}

def sendGoogleChatNotification(String message) {
    node('built-in') {
        def payload = """
        {
            "text": "${message.replace('"', '\\"').replace('\n', '\\n')}"
        }
        """
        
        // Use bat for curl, assuming curl is installed
        bat """
            curl -X POST ^
            -H "Content-Type: application/json" ^
            -d "${payload}" ^
            "%GOOGLE_CHAT_WEBHOOK%" || echo "Notification failed"
        """
    }
}
