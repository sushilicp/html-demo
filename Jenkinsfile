pipeline {
    agent any
    
    environment {
        DOCKER_HOST = "unix:///var/run/docker.sock"
        CONTAINER_NAME = "my-web-app-${env.BUILD_NUMBER}"
        GOOGLE_CHAT_WEBHOOK = "https://chat.googleapis.com/v1/spaces/AAQAaQR_SNA/messages?key=AIzaSyDdI0hCZtE6vySjMm-WEfRq3CPzqKqqsHI&token=RR8wTSfb0py5U2VnLa53xYIJp2yYxVSWV4wP4ovXPxk"
        DEPLOYMENT_URL = "http://localhost:8080"  // Update this
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', 
                url: 'https://github.com/your-repo.git',
                credentialsId: 'your-git-credentials'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh """
                        if docker image inspect my-web-app:${env.BUILD_ID} &>/dev/null; then
                            docker rmi my-web-app:${env.BUILD_ID} || true
                        fi
                    """
                    docker.build("my-web-app:${env.BUILD_ID}", "--no-cache --pull .")
                }
            }
        }

        stage('Run Container') {
            steps {
                script {
                    sh """
                        if docker container inspect ${CONTAINER_NAME} &>/dev/null; then
                            docker stop ${CONTAINER_NAME} || true
                            docker rm ${CONTAINER_NAME} || true
                        fi
                    """
                    
                    docker.image("my-web-app:${env.BUILD_ID}").run(
                        "--name ${CONTAINER_NAME} " +
                        "-p 8080:80 " +
                        "-d " +
                        "--health-cmd 'curl --fail http://localhost:80 || exit 1' " +
                        "--health-interval 5s"
                    )
                    
                    timeout(time: 1, unit: 'MINUTES') {
                        waitUntil {
                            def health = sh(
                                script: "docker inspect --format='{{.State.Health.Status}}' ${CONTAINER_NAME}",
                                returnStdout: true
                            ).trim()
                            return health == "healthy"
                        }
                    }
                }
            }
        }
        
        stage('Smoke Test') {
            steps {
                script {
                    sh "curl -sSf ${DEPLOYMENT_URL} > /dev/null"
                }
            }
        }
    }

    post {
        always {
            script {
                // Capture logs before cleanup
                def containerLogs = sh(
                    script: "docker logs --tail 50 ${CONTAINER_NAME} 2>&1 || true",
                    returnStdout: true
                ).trim()
                
                // Cleanup
                sh """
                    if docker container inspect ${CONTAINER_NAME} &>/dev/null; then
                        docker stop ${CONTAINER_NAME} || true
                        docker rm ${CONTAINER_NAME} || true
                    fi
                """
                cleanWs()
                
                // Store logs for notifications
                currentBuild.containerLogs = containerLogs
            }
        }
        success {
            script {
                def message = """
                🚀 *Deployment Successful* 
                *Build*: #${env.BUILD_NUMBER}
                *Application*: ${env.JOB_NAME}
                *Status*: ${currentBuild.currentResult}
                *Access URL*: ${DEPLOYMENT_URL}
                *Build Logs*: ${env.BUILD_URL}console
                """
                sendGoogleChatNotification(message)
            }
        }
        failure {
            script {
                def message = """
                🔴 *Deployment Failed* 
                *Build*: #${env.BUILD_NUMBER}
                *Application*: ${env.JOB_NAME}
                *Status*: ${currentBuild.currentResult}
                *Last Logs*: ${currentBuild.containerLogs}
                *Build Logs*: ${env.BUILD_URL}console
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
        '${GOOGLE_CHAT_WEBHOOK}' || echo "Google Chat notification failed"
    """
}
