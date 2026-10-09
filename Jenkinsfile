pipeline {
    agent any

    environment {
        JAVA_HOME = '/usr/lib/jvm/java-17-openjdk-amd64'
        PATH = "${JAVA_HOME}/bin:${env.PATH}"

        BACKEND_HOST  = '10.0.1.166'
        FRONTEND_HOST = '10.0.1.18'
        APP_DIR       = '/home/ubuntu/TodoSummaryAssistant'
        IMAGE_TAG     = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Backend Build & Test') {
            steps {
                dir('Backend/todo-summary-assistant') {
                    sh 'mvn clean test'
                }
            }
        }

        stage('Frontend Build') {
            steps {
                dir('Frontend/todo') {
                    sh 'CI=false npm run build'
                    sh 'CI=false npm run build'
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build \
                      -t todo-backend:${IMAGE_TAG} \
                      Backend/todo-summary-assistant

                    docker build \
                      -t todo-frontend:${IMAGE_TAG} \
                      Frontend/todo
                '''
            }
        }

        stage('Deploy Backend') {
            steps {
                sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${BACKEND_HOST} '
                        cd ${APP_DIR} &&
                        git fetch origin main &&
                        git reset --hard origin/main &&

                        cd Backend/todo-summary-assistant &&

                        docker build -t todo-backend:${IMAGE_TAG} . &&

                        docker rm -f todo-backend 2>/dev/null || true &&

                        docker run -d \
                          --name todo-backend \
                          -p 8080:8080 \
                          --restart unless-stopped \
                          --env-file /home/ubuntu/todo-backend.env \
                          todo-backend:${IMAGE_TAG}
                    '
                """
            }
        }

        stage('Deploy Frontend') {
            steps {
                sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${FRONTEND_HOST} '
                        cd ${APP_DIR} &&
                        git fetch origin main &&
                        git reset --hard origin/main &&

                        cd Frontend/todo &&

                        docker build -t todo-frontend:${IMAGE_TAG} . &&

                        docker rm -f todo-frontend 2>/dev/null || true &&

                        docker run -d \
                          --name todo-frontend \
                          -p 80:80 \
                          --restart unless-stopped \
                          todo-frontend:${IMAGE_TAG}
                    '
                """
            }
        }

        stage('Health Check') {
            steps {
                sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${BACKEND_HOST} \
                      'curl -fsS http://localhost:8080/api/todos'

                    ssh -o StrictHostKeyChecking=no ubuntu@${FRONTEND_HOST} \
                      'curl -fsS http://localhost/api/todos'
                """
            }
        }
    }

    post {
        success {
            echo 'Deployment completed successfully!'
        }

        failure {
            echo 'Pipeline failed. Check the failed stage and logs.'
        }
    }
}