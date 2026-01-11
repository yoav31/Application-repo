pipeline {
    agent any 

    environment {
        
        AWS_ACCOUNT_ID = '992382545251'
        AWS_REGION     = 'us-east-1'
        ECR_REPO_NAME  = 'calculator-app'
        IMAGE_URL      = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}"
    }

    stages {
        stage('Unit Tests') {
            agent {
              
                docker { 
                    image 'python:3.11-slim'
                    reuseNode true 
                }
            }
            steps {
                echo 'Running Unit Tests...'
                sh 'python3 -m unittest discover tests'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Building Application Image...'
                script {
                    def tag = "build-${env.BUILD_NUMBER}"
                    sh "docker build -t ${IMAGE_URL}:${tag} ."
                    sh "docker tag ${IMAGE_URL}:${tag} ${IMAGE_URL}:latest"
                }
            }
        }

        stage('Push to ECR') {
            steps {
                echo 'Pushing to Amazon ECR...'
                script {
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                    
                    sh "docker push ${IMAGE_URL}:build-${env.BUILD_NUMBER}"
                    sh "docker push ${IMAGE_URL}:latest"
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline finished successfully!'
        }
        failure {
            echo 'Pipeline failed. Check logs for details.'
        }
    }
}
