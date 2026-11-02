
pipeline {

    agent any

    environment {
        IMAGE_NAME = "python-app"
        CONTAINER_NAME = "python-app"
        HOSTING_IP = "10.0.1.226"
    }

    stages {

        stage('Git Checkout') {
            steps {
                echo 'Downloading application from GitHub'

                checkout scm
            }
        }

        stage('Docker Build') {
            steps {
                echo 'Building Docker Image'

                sh '''
                    docker build \
                    -t python-app:${BUILD_NUMBER} .
                '''
            }
        }

        stage('Docker Image Transfer') {
            steps {

                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'hosting-ssh-key',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {

                    sh '''#!/bin/bash
                        set -euo pipefail

                        echo "Transferring Docker image to hosting server"

                        docker save python-app:${BUILD_NUMBER} \
                        | gzip -1 \
                        | ssh \
                            -i "$SSH_KEY" \
                            -o StrictHostKeyChecking=yes \
                            "$SSH_USER@$HOSTING_IP" \
                            'gunzip | sudo docker load'

                        echo "Docker image transferred successfully"
                    '''
                }
            }
        }

     
        stage('Deploy Application') {
            steps {

                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'hosting-ssh-key',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {

                    sh '''
                        set -euo pipefail

                        echo "Deploying application on hosting server"

                        ssh \
                        -i "$SSH_KEY" \
                        -o StrictHostKeyChecking=yes \
                        "$SSH_USER@$HOSTING_IP" \
                        'sudo docker rm -f python-app || true; sudo docker run -d --restart unless-stopped --name python-app -p 8000:8000 python-app:'"$BUILD_NUMBER"

                        echo "Application deployment completed"
                    '''
                }
            }
        }


  
    stage('Health Check') {
        steps {

            withCredentials([
                sshUserPrivateKey(
                    credentialsId: 'hosting-ssh-key',
                    keyFileVariable: 'SSH_KEY',
                    usernameVariable: 'SSH_USER'
                )
            ]) {

                sh '''
                    set -euo pipefail

                    echo "Waiting for application to start"

                    sleep 5

                    ssh \
                    -i "$SSH_KEY" \
                    -o StrictHostKeyChecking=yes \
                    "$SSH_USER@$HOSTING_IP" \
                    'curl -fsS http://localhost:8000/health'

                    echo "Application health check successful"
                '''
            }
        }
    }

    }

    post {
        success {
            echo 'CI/CD Pipeline completed successfully!'
        }

        failure {
            echo 'CI/CD Pipeline failed. Check console output.'
        }
    }
}
