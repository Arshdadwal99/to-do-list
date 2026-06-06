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
        APP_PORT = '8000'
        PUBLIC_PORT = '80'
        EC2_HOST = '50.19.147.209'
        EC2_INSTANCE_ID = 'i-0036b44d0c4157dc0'
        AWS_REGION = 'us-east-1'
        HEALTH_URL = 'http://50.19.147.209/'
        DEPLOYMENT_TRANSPORT = 'ssm'
        DOCKER_HUB_CREDENTIALS_ID = 'dockerhub-credentials'
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
                echo 'EC2 deployment is executed by DevOpsHub through AWS Systems Manager after the image push completes.'
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                    if [ "$DEPLOYMENT_TRANSPORT" = "ssm" ]; then
                        echo "Health check is executed by DevOpsHub after AWS SSM starts the EC2 container."
                        exit 0
                    fi
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
