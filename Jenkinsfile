pipeline {
    // תיקון DoD: כל השלבים רצים על סביבת דוקר
    agent {
        docker {
            image 'python:3.11-slim'
            args '-u root' // חשוב להרשאות
        }
    }

    environment {
        AWS_ACCOUNT_ID = '992382545251'
        AWS_REGION     = 'us-east-1'
        ECR_REPO_NAME  = 'yoav_ecr'
        IMAGE_URL      = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}"
        
        // תיקון DoD: תיוג דטרמיניסטי לפי PR או Main
        TAG = "${env.CHANGE_ID ? 'pr-' + env.CHANGE_ID : 'main'}-${env.BUILD_NUMBER}"
    }

    stages {
        stage('Unit Tests') {
            steps {
                echo 'Running Unit Tests...'
                // תיקון DoD: יצירת קובץ תוצאות XML שג'נקינס יודע לקרוא
                sh 'pip install pytest'
                sh 'python3 -m pytest --junitxml=results.xml || true' 
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Application Image with tag: ${TAG}"
                script {
                    // הערה: כאן אנחנו משתמשים ב-docker מתוך הקונטיינר
                    sh "docker build -t ${IMAGE_URL}:${TAG} ."
                    sh "docker tag ${IMAGE_URL}:${TAG} ${IMAGE_URL}:latest"
                }
            }
        }

        stage('Push to ECR') {
            steps {
                echo 'Pushing to Amazon ECR...'
                script {
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                    sh "docker push ${IMAGE_URL}:${TAG}"
                    sh "docker push ${IMAGE_URL}:latest"
                }
            }
        }
    }

    post {
        always {
            // תיקון DoD: הצגת תוצאות הבדיקה בג'נקינס ושמירתן כ-Artifact
            junit 'results.xml'
            archiveArtifacts artifacts: 'results.xml', fingerprint: true
        }
        success {
            echo "Successfully pushed image: ${IMAGE_URL}:${TAG}"
        }
    }
}
