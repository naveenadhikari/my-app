pipeline {
    agent none

    environment {
        IMAGE_NAME     = 'my-jenkinsapp'
        CONTAINER_NAME = 'my-jenkinsapp-running'
        DOCKER_USERNAME = credentials('docker-username')
        SERVER_IP      = credentials('server-ip')
        DOCKER_CREDS   = credentials('d97fb965-0bf3-4145-9d57-0d2a21a6c10d')
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
                sh 'docker build -t $DOCKER_USERNAME/$IMAGE_NAME:latest .'
            }
        }

        stage('Push to Docker Hub') {
            agent any
            steps {
                sh '''
                    echo $DOCKER_CREDS_PSW | docker login -u $DOCKER_CREDS_USR --password-stdin
                    docker push $DOCKER_USERNAME/$IMAGE_NAME:latest
                '''
            }
        }

        stage('Deploy to Server') {
            agent any
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'aws-server',
                    keyFileVariable: 'SSH_KEY'
                )]) {
                    sh 'ssh -i $SSH_KEY -o StrictHostKeyChecking=no ubuntu@$SERVER_IP "docker pull $DOCKER_USERNAME/$IMAGE_NAME:latest && docker stop $CONTAINER_NAME || true && docker rm $CONTAINER_NAME || true && docker run -d --name $CONTAINER_NAME --restart unless-stopped -p 3000:3000 $DOCKER_USERNAME/$IMAGE_NAME:latest"'
                }
            }
        }

        stage('Health Check') {
            agent any
            steps {
                script {
                    echo 'Waiting for app to start...'
                    sh 'sleep 10'

                    withCredentials([sshUserPrivateKey(
                        credentialsId: 'aws-server',
                        keyFileVariable: 'SSH_KEY'
                    )]) {
                        def health = sh(
                            script: 'ssh -i $SSH_KEY -o StrictHostKeyChecking=no ubuntu@$SERVER_IP "curl -s http://localhost:3000/health"',
                            returnStdout: true
                        ).trim()

                        echo "Health response: ${health}"
                        env.HEALTH_RESPONSE = health
                    }
                }
            }
        }
    }

    post {

        success {
            node('') {
                withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_URL')]) {
                    script {
                        def duration = currentBuild.durationString.replace(' and counting', '')
                        def appUrl   = "http://${env.SERVER_IP}:3000"
                        def health   = env.HEALTH_RESPONSE ?: 'N/A'

                        def payload = """
                        {
                            "blocks": [
                                {
                                    "type": "header",
                                    "text": { "type": "plain_text", "text": "✅ Deployment Successful" }
                                },
                                {
                                    "type": "section",
                                    "fields": [
                                        { "type": "mrkdwn", "text": "*Job:*\\n${env.JOB_NAME}" },
                                        { "type": "mrkdwn", "text": "*Build:*\\n#${env.BUILD_NUMBER}" },
                                        { "type": "mrkdwn", "text": "*Branch:*\\n${env.GIT_BRANCH}" },
                                        { "type": "mrkdwn", "text": "*Duration:*\\n${duration}" },
                                        { "type": "mrkdwn", "text": "*Health Check:*\\n${health}" },
                                        { "type": "mrkdwn", "text": "*App URL:*\\n${appUrl}" }
                                    ]
                                },
                                {
                                    "type": "actions",
                                    "elements": [
                                        {
                                            "type": "button",
                                            "text": { "type": "plain_text", "text": "View Build Logs" },
                                            "url": "${env.BUILD_URL}console"
                                        }
                                    ]
                                }
                            ]
                        }
                        """

                        sh """
                            curl -s -X POST \$SLACK_URL \
                            -H 'Content-type: application/json' \
                            --data '${payload}'
                        """
                    }
                }
            }
        }

        failure {
            node('') {
                withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_URL')]) {
                    script {
                        def duration   = currentBuild.durationString.replace(' and counting', '')
                        def failedStage = currentBuild.result ?: 'UNKNOWN'

                        def payload = """
                        {
                            "blocks": [
                                {
                                    "type": "header",
                                    "text": { "type": "plain_text", "text": "❌ Deployment Failed" }
                                },
                                {
                                    "type": "section",
                                    "fields": [
                                        { "type": "mrkdwn", "text": "*Job:*\\n${env.JOB_NAME}" },
                                        { "type": "mrkdwn", "text": "*Build:*\\n#${env.BUILD_NUMBER}" },
                                        { "type": "mrkdwn", "text": "*Branch:*\\n${env.GIT_BRANCH}" },
                                        { "type": "mrkdwn", "text": "*Duration:*\\n${duration}" },
                                        { "type": "mrkdwn", "text": "*Status:*\\n${failedStage}" },
                                        { "type": "mrkdwn", "text": "*Triggered by:*\\n${env.BUILD_USER_ID ?: 'SCM / Timer'}" }
                                    ]
                                },
                                {
                                    "type": "section",
                                    "text": { "type": "mrkdwn", "text": "🔍 *Check the console logs to find the exact failed stage and error.*" }
                                },
                                {
                                    "type": "actions",
                                    "elements": [
                                        {
                                            "type": "button",
                                            "text": { "type": "plain_text", "text": "View Failure Logs" },
                                            "url": "${env.BUILD_URL}console"
                                        }
                                    ]
                                }
                            ]
                        }
                        """

                        sh """
                            curl -s -X POST \$SLACK_URL \
                            -H 'Content-type: application/json' \
                            --data '${payload}'
                        """
                    }
                }
            }
        }

        always {
            echo "Pipeline finished — Branch: ${env.GIT_BRANCH} | Build: #${env.BUILD_NUMBER}"
        }
    }
}
