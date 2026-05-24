pipeline {
    agent none  

    environment {
        IMAGE_NAME = 'my-app'
        CONTAINER_NAME = 'my-app-running'
    }

    stages {

        stage('Checkout') {
            agent any
            steps {
                echo 'Pulling latest code...'
                checkout scm
            }
        }

        stage('Install & Test') {
            agent {
                docker {
                    image 'node:18-alpine'
                    args '-u root'  
                }
            }
            steps {
                sh 'npm install'
                sh 'npm test'
            }
        }

        stage('Build Docker image') {
            agent any
            steps {
                sh 'docker build -t $IMAGE_NAME .'
            }
        }

        stage('Deploy') {
            agent any
            steps {
                sh '''
                    docker stop $CONTAINER_NAME || true
                    docker rm $CONTAINER_NAME || true
                    docker run -d --name $CONTAINER_NAME -p 3000:3000 $IMAGE_NAME
                '''
            }
        }

    }

    post {
        success { echo 'Pipeline succeeded! App is live.' }
        failure { echo 'Pipeline failed. Check the logs above.' }
    }
}