pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/sushilicp/html-demo.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("my-web-app:${env.BUILD_ID}", ".")
                }
            }
        }

        stage('Run Container') {
            steps {
                script {
                    docker.image("my-web-app:${env.BUILD_ID}").run("--name my-web-app -p 8080:80 -d")
                }
            }
        }
    }

    post {
        success {
            echo 'Docker container deployed successfully!'
        }
        failure {
            echo 'Pipeline failed! Check logs.'
        }
    }
}
