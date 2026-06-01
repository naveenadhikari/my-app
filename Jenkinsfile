pipeline {
    agent none

    environment {
        IMAGE_NAME      = 'my-jenkinsapp'
        CONTAINER_NAME  = 'my-jenkinsapp-running'
        DOCKER_USERNAME = credentials('docker-username')
        SERVER_IP       = credentials('server-ip')
        DOCKER_CREDS    = credentials('d97fb965-0bf3-4145-9d57-0d2a21a6c10d')
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

        stage('Build Docker Image') {
            agent any
            steps {
                sh "docker build -t ${DOCKER_USERNAME}/${IMAGE_NAME}:latest ."
            }
        }

        stage('Push to Docker Hub') {
            agent any
            steps {
               
                sh """
                    echo $DOCKER_CREDS_PSW | docker login -u $DOCKER_CREDS_USR --password-stdin
                    docker push ${DOCKER_USERNAME}/${IMAGE_NAME}:latest
                """
            }
        }

        stage('Deploy to Server') {
            agent any
            steps {
                echo 'Deploying to production server...'
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'aws-server',
                    keyFileVariable: 'SSH_KEY'
                )]) {
                    sh """
                        ssh -i $SSH_KEY \
                            -o StrictHostKeyChecking=no \ // Disable host key checking for simplicity
                            ubuntu@${SERVER_IP} "
                                docker pull ${DOCKER_USERNAME}/${IMAGE_NAME}:latest &&
                                docker stop ${CONTAINER_NAME} || true &&
                                docker rm ${CONTAINER_NAME} || true &&
                                docker run -d \
                                    --name ${CONTAINER_NAME} \
                                    --restart unless-stopped \
                                    -p 3000:3000 \
                                    ${DOCKER_USERNAME}/${IMAGE_NAME}:latest
                        "
                    """
                }
                echo "App live at http://${SERVER_IP}:3000"
            }
        }

    }

    post {
        success {
            echo 'Pipeline succeeded! App is live on server.'
        }
        failure {
            echo 'Pipeline failed. Check the logs above.'
        }
    }
}