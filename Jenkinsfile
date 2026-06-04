pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '20'))
        disableConcurrentBuilds()
        timeout(time: 45, unit: 'MINUTES')
        timestamps()
    }

    environment {
        PROJECT_TYPE = 'Node.js'
        REPOSITORY = 'Arshdadwal99/to-do-list'
        BRANCH_NAME = 'master'
        DOCKER_IMAGE = 'arshdadwal99/to-do-list'
        CONTAINER_NAME = 'to-do-list'
        APP_PORT = '3000'
        EC2_HOST = '184.72.76.179'
        EC2_USER = 'ubuntu'
        HEALTH_URL = 'http://184.72.76.179:3000/'
        DOCKER_HUB_CREDENTIALS_ID = 'dockerhub-credentials'
        EC2_SSH_CREDENTIALS_ID = 'ec2-ssh-key'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''npm ci'''
            }
        }

        stage('Run Tests') {
            steps {
                sh '''npm test -- --watch=false'''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                        --pull \
                        --label org.opencontainers.image.source=$REPOSITORY \
                        -t $DOCKER_IMAGE:$BUILD_NUMBER \
                        -t $DOCKER_IMAGE:latest \
                        .
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: env.DOCKER_HUB_CREDENTIALS_ID,
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_TOKEN'
                )]) {
                    sh '''
                        echo "$DOCKER_TOKEN" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push $DOCKER_IMAGE:$BUILD_NUMBER
                        docker push $DOCKER_IMAGE:latest
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent(credentials: [env.EC2_SSH_CREDENTIALS_ID]) {
                    withCredentials([usernamePassword(
                        credentialsId: env.DOCKER_HUB_CREDENTIALS_ID,
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )]) {
                        sh '''
                            ssh -o StrictHostKeyChecking=no $EC2_USER@$EC2_HOST "mkdir -p ~/devops-hub/$CONTAINER_NAME"
                            echo "$DOCKER_TOKEN" | ssh -o StrictHostKeyChecking=no $EC2_USER@$EC2_HOST "docker login -u '$DOCKER_USER' --password-stdin"
                            ssh -o StrictHostKeyChecking=no $EC2_USER@$EC2_HOST "docker pull $DOCKER_IMAGE:$BUILD_NUMBER"
                            ssh -o StrictHostKeyChecking=no $EC2_USER@$EC2_HOST "docker rm -f $CONTAINER_NAME || true"
                            ssh -o StrictHostKeyChecking=no $EC2_USER@$EC2_HOST "docker run -d --restart unless-stopped --name $CONTAINER_NAME -p $APP_PORT:$APP_PORT $DOCKER_IMAGE:$BUILD_NUMBER"
                        '''
                    }
                }
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                    for attempt in $(seq 1 30); do
                        if curl -fsS $HEALTH_URL >/dev/null; then
                            echo "Application is healthy at $HEALTH_URL"
                            exit 0
                        fi
                        echo "Waiting for health check ($attempt/30)..."
                        sleep 5
                    done
                    echo "Health check failed for $HEALTH_URL"
                    exit 1
                '''
            }
        }
    }

    post {
        always {
            sh 'docker image prune -f || true'
        }
        success {
            echo 'Jenkins Pipeline Generated and deployment completed successfully.'
        }
        failure {
            echo 'Jenkins pipeline failed. Review console logs for the failed stage.'
        }
    }
}
