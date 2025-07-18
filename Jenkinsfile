pipeline {
    agent any
    
    environment {
        DOCKER_HOST = "unix:///var/run/docker.sock"
        // DOCKER_HOST = "npipe:////./pipe/docker_engine"  # For Windows
    }

    stages {
        stage('Build') {
            steps {
                script {
                    docker.build("my-web-app:${env.BUILD_ID}")
                }
            }
        }
    }
}
