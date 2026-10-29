
pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
    }

    environment {
        DOCKER_USER = 'hoangtrieu123'
        DOCKER_IMAGE = "${DOCKER_USER}/devops-lab8-app"
        DOCKER_TAG = "${BUILD_NUMBER}"
        CONTAINER_NAME = 'lab8-cd-production'
        APP_PORT = '3002'
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'STEP 1: Checkout source code'
                checkout scm
            }
        }

        stage('Docker Build') {
            steps {
                echo 'STEP 2: Build Docker image'

                sh '''
                    docker build \
                        -t "${DOCKER_IMAGE}:${DOCKER_TAG}" \
                        -t "${DOCKER_IMAGE}:latest" \
                        .
                '''
            }
        }

        stage('Docker Scan') {
            steps {
                echo 'STEP 3: Scan image'

                sh '''
                    if docker scout version >/dev/null 2>&1; then
                        docker scout quickview \
                            "${DOCKER_IMAGE}:${DOCKER_TAG}" \
                            || echo "WARNING: Scan failed"
                    else
                        echo "SKIP: Docker Scout not installed"
                    fi
                '''
            }
        }

        stage('Push to Registry') {
            steps {
                echo 'STEP 4: Push Docker Hub'

                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-hub-credentials_1',
                        usernameVariable: 'HUB_USER',
                        passwordVariable: 'HUB_TOKEN'
                    )
                ]) {
                    sh '''
                        set +x

                        printf '%s' "$HUB_TOKEN" | \
                            docker login \
                            -u "$HUB_USER" \
                            --password-stdin

                        docker push "${DOCKER_IMAGE}:${DOCKER_TAG}"
                        docker push "${DOCKER_IMAGE}:latest"

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy Container') {
            steps {
                echo 'STEP 5: Deploy container'

                sh '''
                    docker rm -f "${CONTAINER_NAME}" \
                        2>/dev/null || true

                    docker run -d \
                        --name "${CONTAINER_NAME}" \
                        -p "${APP_PORT}:3000" \
                        --restart unless-stopped \
                        "${DOCKER_IMAGE}:${DOCKER_TAG}"
                '''
            }
        }

        stage('Verify Deploy') {
            steps {
                echo 'STEP 6: Verify deployment'

                sh '''
                    for i in 1 2 3 4 5 6 7 8 9 10; do
                        if curl -fsS \
                            "http://host.docker.internal:${APP_PORT}/health"; then

                            echo ""
                            echo "Deployment verification passed"
                            exit 0
                        fi

                        sleep 2
                    done

                    echo "Deployment verification failed"
                    exit 1
                '''
            }
        }
    }

    post {
        success {
            echo 'CD PIPELINE SUCCESS!'
        }

        failure {
            echo 'CD PIPELINE FAILED!'
        }
    }
}
