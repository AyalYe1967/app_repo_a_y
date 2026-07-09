pipeline {
    agent {
        docker {
            image 'python:3.9-slim'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    
    environment {
        REPOSITORY_NAME = 'a_y/cd'
        // יצירת תג ייחודי המבוסס על מספר הבילד הנוכחי בג'נקינס
        IMAGE_TAG       = "v-${env.BUILD_NUMBER}"
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

        stage('Build Docker Image with Tags') {
            steps {
                script {
                    echo "Building Docker image with tag ${IMAGE_TAG} and latest..."
                    // בניה ראשונית עם התג הייחודי
                    sh "docker build --no-cache -t ${REPOSITORY_NAME}:${IMAGE_TAG} ."
                    // יצירת תג נוסף (Alias) בתור latest
                    sh "docker tag ${REPOSITORY_NAME}:${IMAGE_TAG} ${REPOSITORY_NAME}:latest"
                }
            }
        }
    }
    
    post {
        success {
            echo 'Build and tagging completed successfully!'
        }
        failure {
            echo 'Build stage failed!'
        }
    }
}