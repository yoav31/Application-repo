pipeline {
    agent none 

    environment {
        AWS_ACCOUNT_ID = '992382545251'
        AWS_REGION     = 'us-east-1'
        ECR_REPO_NAME  = 'yoav_ecr'
        IMAGE_URL      = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}"
        PROD_SERVER_IP = '54.85.70.35'
        // DoD: Traceability - הגדרת תגית בסיסית שתעודכן בשלבי הריצה
        TAG            = "build-${env.BUILD_NUMBER}"
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
            agent any
            // DoD: CD stages run only on merges to main
            when { branch 'main' } 
            steps {
                script {
                    // יצירת תגית דטרמיניסטית הכוללת את ה-Commit Hash
                    def gitCommit = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    env.DEPLOY_TAG = "main-${env.BUILD_NUMBER}-${gitCommit}"
                    
                    echo "Building and Pushing image with tag: ${env.DEPLOY_TAG}"
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                    sh "docker build -t ${IMAGE_URL}:${env.DEPLOY_TAG} ."
                    sh "docker tag ${IMAGE_URL}:${env.DEPLOY_TAG} ${IMAGE_URL}:latest"
                    sh "docker push ${IMAGE_URL}:${env.DEPLOY_TAG}"
                    sh "docker push ${IMAGE_URL}:latest"
                }
            }
        }

        stage('Deploy to Production') {
            agent any
            when { branch 'main' } // מונע פריסה בשלב ה-CI
            steps {
                echo "Promoting production candidate to ${PROD_SERVER_IP}..."
                sshagent(['production-ssh']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${PROD_SERVER_IP} "
                            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                            docker pull ${IMAGE_URL}:latest
                            docker stop calculator-app || true
                            docker rm calculator-app || true
                            # DoD: Map host port -> container port
                            docker run -d --name calculator-app -p 5000:5000 ${IMAGE_URL}:latest
                        "
                    """
                }
            }
        }

        stage('Health Verification') {
            agent any
            when { branch 'main' }
            steps {
                script {
                    // DoD: Non-flaky verification with retries/backoff
                    // ה-Pipeline ייכשל אם השרת לא יגיב ב-200 OK
                    retry(5) {
                        echo "Probing service health at http://${PROD_SERVER_IP}:5000/health..."
                        sleep 15
                        sh "curl -f http://${PROD_SERVER_IP}:5000/health"
                    }
                }
            }
        }
    } // סגירת stages

    post {
        success {
            echo "Successfully deployed! Access app at http://${PROD_SERVER_IP}:5000/health"
        }
        failure {
            echo "Pipeline failed. Check ECR credentials or application logs."
        }
    }
}
