pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        APP_NAME = 'acordovadev/proyecto-backend'
        DEPLOY_PATH = 'código/cordova-alumno'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build Image') {
            steps {
                script {
                    dockerImage = docker.build("${APP_NAME}:latest")
                }
            }
        }

        stage('Push Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-credentials') {
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                sh """
                    mkdir -p /opt/${DEPLOY_PATH}
                    cp docker-compose.yml /opt/${DEPLOY_PATH}/
                    cd /opt/${DEPLOY_PATH}
                    docker-compose down
                    docker-compose up -d
                """
            }
        }
    }
}
