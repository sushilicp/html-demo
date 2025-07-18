pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                script {
                    docker.withRegistry('', '') {
                        docker.build("my-web-app:${env.BUILD_ID}").inside {
                            // Your build steps
                        }
                    }
                }
            }
        }
    }
}
