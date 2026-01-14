pipeline {
    agent none 

    environment {
        AWS_ACCOUNT_ID = '992382545251'
        AWS_REGION     = 'us-east-1'
        ECR_REPO_NAME  = 'yoav_ecr'
        IMAGE_URL      = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}"
        // DoD: Traceability - תיוג שמאפשר לעקוב אחרי הקומיט המדויק
        TAG            = "build-${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
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
                sh 'pip install pytest flask'
                sh 'python3 -m pytest --junitxml=results.xml'
            }
            post {
                always {
                    junit 'results.xml'
                }
            }
        }

        stage('Build & Push to ECR') {
            // הפרדה: בנייה ודחיפה של גרסת Production רק ב-Master/Main
            when { branch 'main' } 
            agent any 
            steps {
                script {
                    sh "docker build -t ${IMAGE_URL}:${TAG} ."
                    sh "docker tag ${IMAGE_URL}:${TAG} ${IMAGE_URL}:latest"
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                    sh "docker push ${IMAGE_URL}:${TAG}"
                    sh "docker push ${IMAGE_URL}:latest"
                }
            }
        }

        stage('Deploy to Production') {
            when { branch 'main' } // מונע פריסה במהלך CI של ענפי בדיקה
            agent any
            steps {
                sshagent(['production-ssh']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${PROD_SERVER_IP} "
                            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                            docker pull ${IMAGE_URL}:${TAG}
                            docker stop calculator-app || true
                            docker rm calculator-app || true
                            # DoD: Map host port -> container port
                            docker run -d --name calculator-app -p 5000:5000 ${IMAGE_URL}:${TAG}
                        "
                    """
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
