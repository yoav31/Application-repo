pipeline {
    agent none

    environment {
        AWS_ACCOUNT_ID = '992382545251'
        AWS_REGION     = 'us-east-1'
        ECR_REPO_NAME  = 'yoav_ecr'
        IMAGE_URL      = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}"
        TAG            = "${env.CHANGE_ID ? 'pr-' + env.CHANGE_ID : 'main'}-${env.BUILD_NUMBER}"
        
        // כאן אתה צריך להכניס את ה-IP של שרת ה-Production שלך
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
                sh 'pip install pytest'
                sh 'python3 -m pytest --junitxml=results.xml'
            }
            post {
                always {
                    junit 'results.xml'
                }
            }
        }

        stage('Build & Push') {
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
            agent any
            steps {
                // שימוש בפלאגין SSH Agent כדי להתחבר לשרת ה-Production
                sshagent(['production-ssh']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${PROD_SERVER_IP} "
                            # התחברות ל-ECR בשרת ה-Production
                            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                            
                            # עצירת הקונטיינר הישן אם הוא קיים
                            docker stop calculator-app || true
                            docker rm calculator-app || true
                            
                            # משיכת האימג' החדש והרצתו
                            docker pull ${IMAGE_URL}:${TAG}
                            docker run -d --name calculator-app -p 5000:5000 ${IMAGE_URL}:${TAG}
                        "
                    """
                }
            }
        }
    }

    post {
        always {
            node {
                archiveArtifacts artifacts: 'results.xml'
                echo "Deployment Finished. App should be live at http://${PROD_SERVER_IP}:5000"
            }
