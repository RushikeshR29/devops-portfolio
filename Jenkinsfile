pipeline {
    agent any

    environment {
        IMAGE_NAME     = "devops-portfolio"
        AWS_REGION     = "ap-south-1"
        AWS_ACCOUNT_ID = "728616600636"
        ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        ECR_REPO       = "${ECR_REGISTRY}/${IMAGE_NAME}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Verify Files') {
            steps {
                sh '''
                    pwd
                    ls -lrt
                '''
            }
        }

        stage('SonarQube Scan') {
            steps {
                script {
                    def scannerHome = tool 'sonar-scanner'
                    withSonarQubeEnv('sonarqube') {
                        sh "${scannerHome}/bin/sonar-scanner"
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .'
            }
        }

        stage('ECR Login') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'aws-creds',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''
                        aws ecr get-login-password --region ${AWS_REGION} | \
                        docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    '''
                }
            }
        }

        stage('Tag Image') {
            steps {
                sh 'docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${ECR_REPO}:${BUILD_NUMBER}'
            }
        }

        stage('Push To ECR') {
            steps {
                sh 'docker push ${ECR_REPO}:${BUILD_NUMBER}'
            }
        }

        stage('Verify Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'aws-creds',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''
                        aws ecr describe-images \
                          --repository-name ${IMAGE_NAME} \
                          --region ${AWS_REGION}
                    '''
                }
            }
        }
        
        stage('Deploy') {
            steps {
                sh '''
                    docker rm -f devops-app || true
                    docker run -d --name devops-app --restart unless-stopped \
                      -p 80:8080 ${ECR_REPO}:${BUILD_NUMBER}
                    docker image prune -f

                    for i in 1 2 3 4 5 6 7 8 9 10; do
                        if curl -fsS -o /dev/null http://localhost; then
                            echo "Deployed build ${BUILD_NUMBER}"
                            exit 0
                        fi
                        echo "Waiting for the app to start (attempt $i)..."
                        sleep 3
                    done

                    echo "App did not respond. Container logs:"
                    docker logs devops-app
                    exit 1
                '''
            }
        }
        
    post {
        success {
            echo "Pipeline OK. Deployed ${ECR_REPO}:${BUILD_NUMBER}"
        }
        failure {
            echo 'Pipeline Failed'
        }
    }
}
