pipeline {
    agent any
    
    environment {
        def DOCKER_HOST = "npipe:////./pipe/docker_engine" // Windows Docker endpoint
        def DOCKER_IMAGE = "sushilicp/my-web-app"
        def DOCKER_TAG = "${env.BUILD_ID ?: 'latest'}"
        def CONTAINER_NAME = "my-web-app-${env.BUILD_NUMBER}"
        def GOOGLE_CHAT_WEBHOOK = credentials('google-chat-webhook') // Secure webhook
        def DEPLOYMENT_URL = "http://localhost:8088"
        def DOCKER_HUB_CREDENTIALS = credentials('docker-hub-credentials')
        def HOST_PORT = "8088"
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
                        if docker container inspect ${CONTAINER_NAME} >/dev/null 2>&1; then
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
            node('built-in') {
                script {
                    bat """
                        docker logout || exit 0
                        docker ps -a --filter "name=%CONTAINER_NAME%" --format "{{.ID}}" > temp.txt
                        for /f %%i in (temp.txt) do docker rm -f %%i || exit 0
                        del temp.txt || exit 0
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
                        script: """
                            docker container inspect %CONTAINER_NAME% > nul 2>&1
                            if %ERRORLEVEL% == 0 (
                                docker logs --tail 50 %CONTAINER_NAME% 2>&1 || exit 0
                            ) else (
                                echo Container %CONTAINER_NAME% does not exist
                            )
                        """,
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
        // Escape message for JSON and batch
        def escapedMessage = message.replace('"', '""').replace('\n', '^n').replace('%', '%%').replace('&', '^&')
        def payload = """{"text":"${escapedMessage}"}"""
        
        bat """
            curl -X POST ^
            -H "Content-Type: application/json" ^
            -d "${payload}" ^
            "%GOOGLE_CHAT_WEBHOOK%" || echo Notification failed
        """
    }
}
