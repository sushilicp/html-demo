pipeline {
    agent any
    
    environment {
        DOCKER_HOST = "unix:///var/run/docker.sock"
        CONTAINER_NAME = "my-web-app-${env.BUILD_NUMBER}"
        GOOGLE_CHAT_WEBHOOK = "https://chat.googleapis.com/v1/spaces/AAQAaQR_SNA/messages?key=AIzaSyDdI0hCZtE6vySjMm-WEfRq3CPzqKqqsHI&token=RR8wTSfb0py5U2VnLa53xYIJp2yYxVSWV4wP4ovXPxk"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', 
                url: 'https://github.com/sushilicp/html-demo.git',
                credentialsId: ''
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                        sh "docker rmi my-web-app:${env.BUILD_ID} || true"
                    }
                    docker.build("my-web-app:${env.BUILD_ID}", "--no-cache --pull .")
                }
            }
        }

        stage('Run Container') {
            steps {
                script {
                    catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                        sh "docker stop ${CONTAINER_NAME} || true"
                        sh "docker rm ${CONTAINER_NAME} || true"
                    }
                    
                    docker.image("my-web-app:${env.BUILD_ID}").run(
                        "--name ${CONTAINER_NAME} " +
                        "-p 8080:80 " +
                        "-d " +
                        "--health-cmd 'curl --fail http://localhost:80 || exit 1' " +
                        "--health-interval 5s"
                    )
                    
                    sleep(time: 10, unit: 'SECONDS')
                    def health = sh(
                        script: "docker inspect --format='{{.State.Health.Status}}' ${CONTAINER_NAME}",
                        returnStdout: true
                    ).trim()
                    if (health != "healthy") {
                        error "Container failed health check: ${health}"
                    }
                }
            }
        }
        
        stage('Smoke Test') {
            steps {
                script {
                    sh """
                        curl -sSf http://localhost:8080 > /dev/null || \
                        (echo 'Web server not responding' && exit 1)
                    """
                }
            }
        }
    }

    post {
        always {
            script {
                cleanWs()
                sh "docker ps -a --filter 'name=${CONTAINER_NAME}'"
            }
        }
        success {
            script {
                def successMessage = """
                    ✅ *Build #${env.BUILD_NUMBER} Success*
                    *Application*: ${env.JOB_NAME}
                    *Status*: Deployed successfully
                    *Access URL*: http://your-server:8080
                    *Build Log*: ${env.BUILD_URL}console
                """
                sendGoogleChatNotification(successMessage)
            }
        }
        failure {
            script {
                def logs = sh(script: "docker logs ${CONTAINER_NAME} --tail 50 || true", returnStdout: true).trim()
                def failureMessage = """
                    ❌ *Build #${env.BUILD_NUMBER} Failed*
                    *Application*: ${env.JOB_NAME}
                    *Error*: ${currentBuild.currentResult}
                    *Last Logs*: ${logs}
                    *Build Log*: ${env.BUILD_URL}console
                """
                sendGoogleChatNotification(failureMessage)
            }
        }
    }
}

def sendGoogleChatNotification(String message) {
    def payload = """
    {
        "text": "${message.replaceAll('"', '\\\\"').replaceAll('\n', '\\\\n')}"
    }
    """
    
    sh """
        curl -X POST \
        -H 'Content-Type: application/json' \
        -d '${payload}' \
        '${env.GOOGLE_CHAT_WEBHOOK}'
    """
}
