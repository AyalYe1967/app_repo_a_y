pipeline {
    agent {
        docker {
            image 'python:3.9-slim'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    
    environment {
        REPOSITORY_NAME = 'a_y/cd'
    }

    stages {
        stage('Install Docker CLI in Agent') {
            steps {
                script {
                    echo "Ensuring Docker CLI is available in the agent environment..."
                    sh "apt-get update && apt-get install -y docker.io"
                }
            }
        }

        stage('Build Initial Docker Image') {
            steps {
                script {
                    echo "Building temporary Docker image for calculator app..."
                    sh "docker build --no-cache -t ${REPOSITORY_NAME}:latest ."
                }
            }
        }
    }
    
    post {
        success {
            echo 'Build stage completed successfully!'
        }
        failure {
            echo 'Build stage failed!'
        }
    }
}