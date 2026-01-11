pipeline {
    agent none // אנחנו נגדיר אג'נט לכל שלב בנפרד

    environment {
        AWS_ACCOUNT_ID = '992382545251'
        AWS_REGION     = 'us-east-1'
        ECR_REPO_NAME  = 'yoav_ecr'
        IMAGE_URL      = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}"
        TAG            = "${env.CHANGE_ID ? 'pr-' + env.CHANGE_ID : 'main'}-${env.BUILD_NUMBER}"
    }

    stages {
        stage('Unit Tests') {
            agent {
                docker { 
                    image 'python:3.11-slim'
                    args '-u root'
                }
            }
            steps {
                sh 'pip install pytest'
                sh 'python3 -m pytest --junitxml=results.xml'
            }
        }

        stage('Build & Push') {
            agent any // שלב זה רץ על השרת (שיש לו גישה לפקודת docker)
            steps {
                script {
                    // בנייה
                    sh "docker build -t ${IMAGE_URL}:${TAG} ."
                    sh "docker tag ${IMAGE_URL}:${TAG} ${IMAGE_URL}:latest"
                    
                    // התחברות ודחיפה
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                    sh "docker push ${IMAGE_URL}:${TAG}"
                    sh "docker push ${IMAGE_URL}:latest"
                    
                    echo "Image reference: ${IMAGE_URL}:${TAG}"
                }
            }
        }
    }

    post {
        always {
            // שמירת תוצאות הטסטים מהשלב הראשון
            junit 'results.xml'
            archiveArtifacts artifacts: 'results.xml'
        }
    }
}
