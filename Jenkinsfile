pipeline {
    agent none

    environment {
        IMAGE_NAME        = 'my-jenkinsapp'
        CONTAINER_NAME    = 'my-jenkinsapp-running'
        DOCKER_USERNAME   = credentials('docker-username')
        SERVER_IP         = credentials('server-ip')
        DOCKER_CREDS      = credentials('d97fb965-0bf3-4145-9d57-0d2a21a6c10d')
        GEMINI_API_KEY    = credentials('gemini-api-key')
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
            script {
                echo 'Deployment succeeded. Getting AI health report...'

                def info = "Build #${env.BUILD_NUMBER} succeeded. Branch: ${env.GIT_BRANCH}. Health endpoint response: ${env.HEALTH_RESPONSE}. Duration: ${currentBuild.durationString}"

                def response = sh(
                    script: """
                        curl -s "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=\$GEMINI_API_KEY" \
                            -H "content-type: application/json" \
                            -d '{"contents":[{"parts":[{"text":"You are a DevOps assistant. Write a brief 3 sentence deployment health report based on this info. Be professional: ${info}"}]}]}'
                    """,
                    returnStdout: true
                ).trim()

                def aiReport = sh(
                    script: """echo '${response}' | python3 -c "import sys,json; data=json.load(sys.stdin); print(data['candidates'][0]['content']['parts'][0]['text'])" """,
                    returnStdout: true
                ).trim()

                echo """
===========================================
   DEPLOYMENT SUCCESSFUL
===========================================
AI Health Report:
${aiReport}

App URL: http://${env.SERVER_IP}:3000
Build: #${env.BUILD_NUMBER}
===========================================
                """
            }
        }

        failure {
            script {
                echo 'Pipeline failed. Getting AI analysis...'

                def logs = currentBuild.rawBuild
                    .getLog(80)
                    .join(' ')
                    .replaceAll('"', '')
                    .replaceAll("'", '')
                    .take(2000)

                def response = sh(
                    script: """
                        curl -s "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=\$GEMINI_API_KEY" \
                            -H "content-type: application/json" \
                            -d '{"contents":[{"parts":[{"text":"You are a DevOps expert. Analyze this Jenkins pipeline failure. Give: 1) What failed 2) Root cause 3) How to fix it. Be concise: ${logs}"}]}]}'
                    """,
                    returnStdout: true
                ).trim()

                def aiAnalysis = sh(
                    script: """echo '${response}' | python3 -c "import sys,json; data=json.load(sys.stdin); print(data['candidates'][0]['content']['parts'][0]['text'])" """,
                    returnStdout: true
                ).trim()

                echo """
===========================================
   PIPELINE FAILED
===========================================
AI Failure Analysis:
${aiAnalysis}

Build: #${env.BUILD_NUMBER}
Logs: ${env.BUILD_URL}console
===========================================
                """
            }
        }

        always {
            echo "Pipeline finished - Build #${env.BUILD_NUMBER}"
        }

    }
}