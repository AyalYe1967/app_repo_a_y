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
        IMAGE_TAG       = "${env.CHANGE_ID != null && env.CHANGE_ID != '' ? 'pr-' + env.CHANGE_ID + '-' + env.BUILD_NUMBER : 'v-' + env.BUILD_NUMBER}"
        REGISTRY_URL    = "${env.AWS_ACC_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com"
        
        EC2_HOST        = '34.239.116.186'
        EC2_USER        = 'ubuntu'
        CONTAINER_NAME  = 'my-running-app'
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
                    echo "Running unit and integration tests from tests directory..."
                    sh "docker run --rm -w /app ${REPOSITORY_NAME}:${IMAGE_TAG} python -m unittest tests/test_calculator_logic.py > unit_test_results.txt || (cat unit_test_results.txt && exit 1)"
                    sh "docker run --rm -w /app ${REPOSITORY_NAME}:${IMAGE_TAG} python -m unittest tests/test_calculator_app_integration.py > integration_test_results.txt || (cat integration_test_results.txt && exit 1)"
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
                    withCredentials([usernamePassword(credentialsId: 'aws-access-key', 
                                                    usernameVariable: 'AWS_ACCESS_KEY_ID', 
                                                    passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        
                        sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${REGISTRY_URL}"
                        
                        echo "Tagging image for ECR with ${IMAGE_TAG}..."
                        sh "docker tag ${REPOSITORY_NAME}:${IMAGE_TAG} ${REGISTRY_URL}/${REPOSITORY_NAME}:${IMAGE_TAG}"
                        sh "docker tag ${REPOSITORY_NAME}:${IMAGE_TAG} ${REGISTRY_URL}/${REPOSITORY_NAME}:latest"
                        
                        echo "Pushing image to ECR..."
                        sh "docker push ${REGISTRY_URL}/${REPOSITORY_NAME}:${IMAGE_TAG}"
                        sh "docker push ${REGISTRY_URL}/${REPOSITORY_NAME}:latest"

                        echo "===================================================="
                        echo " PUSHED IMAGE REFERENCE: ${REGISTRY_URL}/${REPOSITORY_NAME}:${IMAGE_TAG}"
                        echo "===================================================="
                    }
                }
            }
        }

        stage('Notify GitHub Success') {
            when {
                expression { (env.CHANGE_ID != null && env.CHANGE_ID != '') || env.BRANCH_NAME == 'main' || env.BRANCH_NAME == 'master' }
            }
            steps {
                script {
                    echo "Notifying GitHub that the build and tests have passed..."
                    githubNotify status: 'SUCCESS', description: 'Build, Tests, ECR & Deploy Passed!', context: 'col_app_pipeline'
                }
            }
        }

        stage('Deploy to Production EC2') {
            when {
                expression { env.BRANCH_NAME == 'main' || env.BRANCH_NAME == 'master' }
            }
            steps {
                script {
                    echo "Deploying the latest image to Production EC2 host..."
                    withCredentials([file(credentialsId: 'b7943e0f-cf0c-4a33-8d0f-eda0073045d8', variable: 'SSH_KEY_FILE')]) {
                        sh """
                            chmod 600 \$SSH_KEY_FILE
                            ssh -i \$SSH_KEY_FILE -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} "\
                                aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${REGISTRY_URL} && \
                                docker pull ${REGISTRY_URL}/${REPOSITORY_NAME}:latest && \
                                docker stop ${CONTAINER_NAME} || true && \
                                docker rm ${CONTAINER_NAME} || true && \
                                docker run -d --name ${CONTAINER_NAME} -p 5000:5000 ${REGISTRY_URL}/${REPOSITORY_NAME}:latest \
                            "
                        """
                    }
                    echo "Deployment completed successfully. New release is now live!"
                }
            }
        }
    }
    
    post {
        always {
            archiveArtifacts artifacts: '*_test_results.txt', allowEmptyArchive: true
        }
        failure {
            script {
                try {
                    githubNotify status: 'FAILURE', description: 'Pipeline failed at some stage!', context: 'cd_practise_multy'
                } catch(Exception e) {
                    echo "Could not send failure notification: ${e.message}"
                }
            }
            echo 'Pipeline failed!'
        }
    }
}