pipeline {
    agent none

    environment {
        AWS_ACCOUNT_ID = '992382545251'
        AWS_REGION     = 'us-east-1'
        ECR_REPO_NAME  = 'yoav_ecr'
        IMAGE_URL      = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}"
        // DoD: תיוג דטרמיניסטי (main או pr)
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
                // DoD: הרצת טסטים ויצירת דוח XML
                sh 'python3 -m pytest --junitxml=results.xml'
            }
            post {
                always {
                    // DoD: הצגת תוצאות בג'נקינס
                    junit 'results.xml'
                }
            }
        }

        stage('Build & Push') {
            agent any 
            steps {
                script {
                    // בניית האימג'
                    sh "docker build -t ${IMAGE_URL}:${TAG} ."
                    sh "docker tag ${IMAGE_URL}:${TAG} ${IMAGE_URL}:latest"
                    
                    // התחברות ודחיפה ל-ECR
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                    sh "docker push ${IMAGE_URL}:${TAG}"
                    sh "docker push ${IMAGE_URL}:latest"
                    
                    // DoD: הדפסת האימג' ללוג
                    echo "Image reference: ${IMAGE_URL}:${TAG}"
                }
            }
        }
    }

    post {
        always {
            node {
                // DoD: שמירת תוצאות כ-Artifact
                archiveArtifacts artifacts: 'results.xml', fingerprint: true
            }
        }
    }
}
