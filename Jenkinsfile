pipeline {
    agent none

    environment {
        AWS_ACCOUNT_ID = '992382545251'
        AWS_REGION     = 'us-east-1'
        ECR_REPO_NAME  = 'yoav_ecr'
        IMAGE_URL      = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}"
        // DoD: תיוג דטרמיניסטי ל-PR או ל-Main
        TAG            = "${env.CHANGE_ID ? 'pr-' + env.CHANGE_ID : 'main'}-${env.BUILD_NUMBER}"
        // החלף ל-IP של שרת ה-Production שלך
        PROD_SERVER_IP = '54.85.70.35' 
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
                echo 'Running Unit Tests inside Docker...'
                sh 'pip install pytest'
                // הרצת טסטים ויצירת דוח XML לצפייה בג'נקינס
                sh 'python3 -m pytest --junitxml=results.xml'
            }
            post {
                always {
                    junit 'results.xml'
                }
            }
        }

        stage('Build Docker Image') {
            agent any
            steps {
                script {
                    echo "Building image: ${IMAGE_URL}:${TAG}"
                    sh "docker build -t ${IMAGE_URL}:${TAG} ."
                    sh "docker tag ${IMAGE_URL}:${TAG} ${IMAGE_URL}:latest"
                }
            }
        }

        stage('Push to ECR') {
            agent any
            steps {
                script {
                    echo 'Pushing image to AWS ECR...'
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                    sh "docker push ${IMAGE_URL}:${TAG}"
                    sh "docker push ${IMAGE_URL}:latest"
                }
            }
        }

        stage('Deploy to Production') {
            agent any
            steps {
                echo "Deploying to Production Server at ${PROD_SERVER_IP}..."
                sshagent(['production-ssh']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${PROD_SERVER_IP} "
                            # התחברות ל-ECR בשרת המרוחק
                            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                            
                            # ניקוי קונטיינרים ישנים
                            docker stop calculator-app || true
                            docker rm calculator-app || true
                            
                            # משיכה והרצה של הגרסה החדשה
                            docker pull ${IMAGE_URL}:${TAG}
                            docker run -d --name calculator-app -p 5000:5000 ${IMAGE_URL}:${TAG}
                        "
                    """
                }
            }
        }
    }

    	stage('Health Verification') {
            when { branch 'main' }
            agent any
            steps {
                script {
                    // DoD: Health check passes with non-flaky verification (retries/backoff)
                    // ה-Pipeline ייכשל אם ה-curl לא יחזיר תשובה תקינה אחרי 5 ניסיונות
                    retry(5) {
                        echo "Probing service health at /health..."
                        sleep 15 // זמן המתנה לעליית הקונטיינר (Backoff)
                        sh "curl -f http://${PROD_SERVER_IP}:5000/health"
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Successfully deployed! Access your app at http://${PROD_SERVER_IP}:5000/health"
        }
        failure {
            echo "Pipeline failed. Deployment rolled back or service is unhealthy."
        }
    }
}
