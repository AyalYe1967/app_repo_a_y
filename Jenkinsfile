pipeline {
    agent {
        docker {
            image 'python:3.9-slim'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    
    environment {
        AWS_REGION      = 'us-east-1' 
        AWS_ACC_ID      = '992382545251' 
        REPOSITORY_NAME = 'a_y/cd'
        IMAGE_TAG       = "v-${env.BUILD_NUMBER}"
        REGISTRY_URL    = "${env.AWS_ACC_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com"
    }

    stages {
        stage('Install Tools in Agent') {
            steps {
                script {
                    echo "Ensuring Docker CLI and AWS CLI are available in the agent environment..."
                    sh "apt-get update && apt-get install -y docker.io awscli"
                }
            }
        }

        stage('Build image') {
            steps {
                script {
                    echo "Building Docker image with tag ${IMAGE_TAG} and latest..."
                    sh "docker build --no-cache -t ${REPOSITORY_NAME}:${IMAGE_TAG} ."
                    sh "docker tag ${REPOSITORY_NAME}:${IMAGE_TAG} ${REPOSITORY_NAME}:latest"
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    echo "Running unit and integration tests from tests directory inside the container..."
                    sh "docker run --rm -w /app ${REPOSITORY_NAME}:${IMAGE_TAG} python -m unittest tests/test_calculator_logic.py"
                    sh "docker run --rm -w /app ${REPOSITORY_NAME}:${IMAGE_TAG} python -m unittest tests/test_calculator_app_integration.py"
                }
            }
        }

        stage('Push image to ECR') {
            when {
                expression { (env.CHANGE_ID != null && env.CHANGE_ID != '') || env.BRANCH_NAME == 'main' || env.BRANCH_NAME == 'master' }
            }
            steps {
                script {
                    echo "Logging in to Amazon ECR using credential id 'aws-access-key'..."
                    // שימוש ב-ID המדויק שהגדרת בג'נקינס
                    withCredentials([usernamePassword(credentialsId: 'aws-access-key', 
                                                    usernameVariable: 'AWS_ACCESS_KEY_ID', 
                                                    passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        
                        sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${REGISTRY_URL}"
                        
                        echo "Tagging image for ECR..."
                        sh "docker tag ${REPOSITORY_NAME}:${IMAGE_TAG} ${REGISTRY_URL}/${REPOSITORY_NAME}:${IMAGE_TAG}"
                        sh "docker tag ${REPOSITORY_NAME}:${IMAGE_TAG} ${REGISTRY_URL}/${REPOSITORY_NAME}:latest"
                        
                        echo "Pushing image to ECR..."
                        sh "docker push ${REGISTRY_URL}/${REPOSITORY_NAME}:${IMAGE_TAG}"
                        sh "docker push ${REGISTRY_URL}/${REPOSITORY_NAME}:latest"
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo 'CI pipeline (Build, Test, Push) completed successfully!'
        }
        failure {
            echo 'CI pipeline failed!'
        }
    }
}