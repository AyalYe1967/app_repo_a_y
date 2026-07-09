pipeline {
    agent {
        docker {
            image 'python:3.9-slim'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    
    environment {
        REPOSITORY_NAME = 'a_y/cd'
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
                    sh "docker build --no-cache -t ${REPOSITORY_NAME}:${IMAGE_TAG} ."
                    sh "docker tag ${REPOSITORY_NAME}:${IMAGE_TAG} ${REPOSITORY_NAME}:latest"
                }
            }
        }

        stage('Run Tests (Unit & Integration)') {
            steps {
                script {
                    echo "Running unit and integration tests inside the container..."
                    // הרצת בדיקות היחידה (Unit Tests)
                    sh "docker run --rm ${REPOSITORY_NAME}:${IMAGE_TAG} python -m unittest test_calculator_logic.py"
                    // הרצת בדיקות האינטגרציה הקלות (Integration Tests)
                    sh "docker run --rm ${REPOSITORY_NAME}:${IMAGE_TAG} python -m unittest test_calculator_app_integration.py"
                }
            }
        }
    }
    
    post {
        success {
            echo 'Build and all tests completed successfully!'
        }
        failure {
            echo 'Build or tests failed!'
        }
    }
}