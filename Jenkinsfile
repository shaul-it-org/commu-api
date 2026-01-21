pipeline {
    agent any

    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'prod'], description: 'Deployment environment')
        choice(name: 'ACTION', choices: ['deploy', 'rollback', 'switch'], description: 'Deployment action (prod only)')
        choice(name: 'TARGET_SLOT', choices: ['auto', 'blue', 'green'], description: 'Target slot for deployment (prod only)')
    }

    environment {
        IMAGE_NAME = 'commu-api'
        ACTIVE_SLOT_FILE = '/opt/commu-api/.active-slot'
    }

    stages {
        stage('Initialize') {
            steps {
                script {
                    if (params.ENVIRONMENT == 'dev') {
                        env.IMAGE_TAG = 'dev'
                        env.CONTAINER_NAME = 'commu-api-dev'
                        env.PORT = '10400'
                        env.SPRING_PROFILE = 'dev'
                        env.DB_NAME = 'commu_dev'
                    } else {
                        env.IMAGE_TAG = 'latest'
                        env.SPRING_PROFILE = 'prod'
                        env.DB_NAME = 'commu'
                        env.BLUE_CONTAINER = 'commu-api-blue'
                        env.GREEN_CONTAINER = 'commu-api-green'
                        env.BLUE_PORT = '10300'
                        env.GREEN_PORT = '10301'

                        def currentSlot = sh(
                            script: "cat ${ACTIVE_SLOT_FILE} 2>/dev/null || echo 'blue'",
                            returnStdout: true
                        ).trim()

                        if (params.TARGET_SLOT == 'auto') {
                            env.DEPLOY_SLOT = currentSlot == 'blue' ? 'green' : 'blue'
                        } else {
                            env.DEPLOY_SLOT = params.TARGET_SLOT
                        }

                        env.CURRENT_SLOT = currentSlot
                        env.CONTAINER_NAME = env.DEPLOY_SLOT == 'blue' ? env.BLUE_CONTAINER : env.GREEN_CONTAINER
                        env.PORT = env.DEPLOY_SLOT == 'blue' ? env.BLUE_PORT : env.GREEN_PORT

                        echo "Current active slot: ${env.CURRENT_SLOT}"
                        echo "Deploying to slot: ${env.DEPLOY_SLOT}"
                    }

                    echo "Environment: ${params.ENVIRONMENT}"
                    echo "Container: ${env.CONTAINER_NAME}"
                    echo "Port: ${env.PORT}"
                }
            }
        }

        stage('Checkout') {
            when {
                expression { params.ACTION == 'deploy' }
            }
            steps {
                checkout scm
            }
        }

        stage('Build') {
            when {
                expression { params.ACTION == 'deploy' }
            }
            steps {
                sh './gradlew clean bootJar --no-daemon'
            }
        }

        stage('Docker Build') {
            when {
                expression { params.ACTION == 'deploy' }
            }
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('Deploy') {
            when {
                expression { params.ACTION == 'deploy' }
            }
            steps {
                sh """
                    docker stop ${CONTAINER_NAME} || true
                    docker rm ${CONTAINER_NAME} || true
                    docker run -d \
                        --name ${CONTAINER_NAME} \
                        --network postgres_default \
                        -p ${PORT}:8080 \
                        -e SPRING_PROFILES_ACTIVE=${SPRING_PROFILE} \
                        -e SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/${DB_NAME} \
                        -e DB_USERNAME=postgres \
                        -e DB_PASSWORD=postgres \
                        --restart unless-stopped \
                        ${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage('Health Check') {
            when {
                expression { params.ACTION == 'deploy' }
            }
            steps {
                script {
                    def maxRetries = 10
                    def retryCount = 0
                    def healthy = false

                    while (retryCount < maxRetries && !healthy) {
                        sleep(time: 5, unit: 'SECONDS')
                        try {
                            def response = sh(
                                script: "curl -s -o /dev/null -w '%{http_code}' http://localhost:${PORT}/api/v1/health",
                                returnStdout: true
                            ).trim()
                            if (response == '200') {
                                healthy = true
                                echo "Health check passed!"
                            }
                        } catch (Exception e) {
                            echo "Health check attempt ${retryCount + 1} failed"
                        }
                        retryCount++
                    }

                    if (!healthy) {
                        error("Health check failed after ${maxRetries} attempts")
                    }
                }
            }
        }

        stage('Switch Traffic') {
            when {
                expression { params.ENVIRONMENT == 'prod' && (params.ACTION == 'deploy' || params.ACTION == 'switch') }
            }
            steps {
                sh "echo '${DEPLOY_SLOT}' > ${ACTIVE_SLOT_FILE}"
                echo "Traffic switched to ${DEPLOY_SLOT} slot"
            }
        }

        stage('Rollback') {
            when {
                expression { params.ENVIRONMENT == 'prod' && params.ACTION == 'rollback' }
            }
            steps {
                script {
                    def rollbackSlot = env.CURRENT_SLOT == 'blue' ? 'green' : 'blue'
                    sh "echo '${rollbackSlot}' > ${ACTIVE_SLOT_FILE}"
                    echo "Rolled back to ${rollbackSlot} slot"
                }
            }
        }
    }

    post {
        success {
            script {
                if (params.ENVIRONMENT == 'dev') {
                    echo "Deployment successful! DEV API available at https://dev-api.shaul.link"
                } else {
                    echo "Deployment successful! PROD API available at https://api.shaul.link"
                    echo "Active slot: ${DEPLOY_SLOT}"
                }
            }
        }
        failure {
            echo "Deployment failed!"
            sh "docker logs ${CONTAINER_NAME} --tail 100 || true"
        }
    }
}
